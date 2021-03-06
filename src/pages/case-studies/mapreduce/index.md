# Case Study: Pygmy Hadoop {#map-reduce}

In this case study we're going to implement
a simple-but-powerful parallel processing framework
using `Monoids`, `Functors`, and a host of other goodies.

If you have used Hadoop or otherwise worked in "big data"
you will have heard of [MapReduce][link-mapreduce],
which is a programming model for doing parallel data processing
across tens or hundreds of machines.
As the name suggests, the model is built around a *map* phase,
which is the same `map` function we know
from Scala and the `Functor` type class,
and a *reduce* phase, which we usually call `fold`[^hadoop-shuffle] in Scala.

[^hadoop-shuffle]: In Hadoop there is also a shuffle phase
that we will ignore here.

## Parallelizing *map* and *fold*

Recall the general signature for `map` is
to apply a function `A => B` to a `F[A]`, returning a `F[B]`:

![Type chart: functor map](src/pages/functors/generic-map.pdf+svg){#fig:mapreduce:functor-type-chart}

`map` transforms each individual element in a sequence independently.
We can easily parallelize `map` because
there are no dependencies between
the transformations applied to different elements
(the type signature of the function `A => B` shows us this,
assuming we don't use side-effects not reflected in the types).

What about `fold`?
We can implement this step with an instance of `Foldable`.
Not every functor also has an instance of foldable,
but we can implement a map reduce system
on top of any data type that has both of these type classes.
Our reduction step becomes a `fold`
over the results of the distributed `map`.

![Type chart: fold](src/pages/foldable-traverse/generic-foldleft.pdf+svg){#fig:mapreduce:foldleft-type-chart}

If you remember from our discussion of `Foldable`,
then depending on the reduction operation we use,
the order of combination can have effect on the final result.
To remain correct we need to ensure
our reduction operation is *associative*:

```scala
reduce(a1, reduce(a2, a3)) == reduce(reduce(a1, a2), a3)
```

If we have associativity,
we can arbitrarily distribute work
amongst threads or machines
provided we preserve the ordering
on the sequence of elements we're processing.

Our fold operation requires us to seed the computation
with an element of type `B`.
Since our fold may be split
into an arbitrary number of parallel steps,
the seed should not effect the result of the computation.
This naturally requires the seed to be an *identity* element:

```scala
reduce(seed, a1) == reduce(a1, seed) == a1
```

In summary, our parallel fold will yield the correct results if:

- we require the reducer function to be associative;
- we seed the computation with the identity of this function.

What does this pattern sound like?
That's right, we've come full circle back to `Monoid`,
the first type class we discussed in this book.
We are not the first to recognise the importance of monoids.
The [monoid design pattern for map-reduce jobs][link-mapreduce-monoid]
is at the core of recent big data systems
such as Twitter's [Summingbird][link-summingbird].

In this project we're going to implement
a very simple single-machine map-reduce.
We'll start by implementing a method called `foldMap`
to model the data-flow we need.
We'll then parallelize `foldMap`
and see how we can introduce error handling
using monads, applicative functors,
and a new tool called natural transformations.
Finally we'll ground our discussion
by looking at some of more interesting monoids
that are applicable for processing large data sets.

## Implementing *foldMap*

Start by implementing a method called `foldMap` that:

 - accepts a sequence parameter of type `Iterable[A]`
   and a function of type `A => B`,
   where there is a `Monoid` for `B`;
 - maps the function over the sequence;
 - reduces the results using the monoid.

Here's a basic type signature.
You will have to add implicit parameters or context bounds
to complete the type signature:

```tut:book:silent
/** Single-threaded map reduce function.
  * Maps `func` over `values`
  * and reduces using a `Monoid[B]`.
  */
def foldMap[A, B](values: Iterable[A])(func: A => B): B =
  ???
```

Here's some sample output:

```tut:book:invisible
import cats.Monoid
import cats.syntax.semigroup._

def foldMap[A, B: Monoid](values: Iterable[A])(func: A => B): B =
  values.foldLeft(Monoid[B].empty)(_ |+| func(_))
```

```tut:book:silent
import cats.instances.int._
```

```tut:book
foldMap(List(1, 2, 3))(identity)

```tut:book:silent
import cats.instances.string._
```

```tut:book
// Mapping to a String uses the concatenation monoid:
foldMap(List(1, 2, 3))(_.toString + "! ")

// Mapping over a String to produce a String:
foldMap("Hello world!")(_.toString.toUpperCase)
```

<div class="solution">
We have to modify the type signature to accept a `Monoid` for `B`.
With that change we can use
the `Monoid` `empty` and `|+|` syntax [described in the monoids chapter](#monoid-syntax):

```tut:book:silent
import cats.Monoid
import cats.instances.int._
import cats.instances.string._
import cats.syntax.semigroup._

def foldMap[A, B : Monoid](values: Iterable[A])(func: A => B = (a: A) => a): B =
  values.map(func).foldLeft(Monoid[B].empty)(_ |+| _)
```

We can make a slight alteration to this code to do everything in one step:

```tut:book:silent
def foldMap[A, B : Monoid](values: Iterable[A])(func: A => B = (a: A) => a): B =
  values.foldLeft(Monoid[B].empty)(_ |+| func(_))
```
</div>

## Parallelising *foldMap*

To run the fold in parallel we need to change our implementation strategy.
A simple strategy is to allocate as many threads as we have CPUs
and evenly partition our sequence amongst the threads.
We can append the results together as each thread completes.

Scala provides some simple tools to distribute work amongst threads.
We could simply use
the [parallel collections library][link-parallel-collections] to implement a solution,
but let's challenge ourselves by diving a bit deeper.
You might have already used `Futures`.
A `Future` models a computation that may not yet have a value.
That is, it represents a value that will become available "in the future".
They are a good tool for this sort of job.

Before we begin we need to introduce some new building blocks:
*futures* and *partioning sequences*.

### Futures

To execute an operation in parallel we can construct a `Future` as follows:

```tut:book:silent
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global
```

```tut:book
val future: Future[String] =
  Future { "Construct this string in parallel!" }
```

We need to have an implicit `ExecutionContext` in scope,
which determines which thread pool runs the operation.
The default `ExecutionContext.Implicits.global` shown above is a good choice,
but we can use any choice in practice.

We operate on the value in a `Future` using the familiar `map` and `flatMap` methods:

```tut:book
val future2 = future.map(_.length)

val future3 =
  future2.flatMap { length => Future { length * 1000 } }
```

If we have a `List[Future[A]]` we can convert it
to a `Future[List[A]]` using the method `Future.sequence`:

```tut:book
Future.sequence(List(Future(1), Future(2), Future(3)))
```

or an instance of `Traverse`:

```tut:book:silent
import cats.instances.future._ // Applicative for Future
import cats.instances.list._   // Traverse for List
import cats.syntax.traverse._  // foo.sequence syntax
```

```tut:book
List(Future(1), Future(2), Future(3)).sequence
```

Finally, we can use `Await.result`
to block on a `Future` till a result is available.

```tut:book:silent
import scala.concurrent._
import scala.concurrent.duration._
```

```tut:book
Await.result(Future(1), 1.second) // wait forever until a result arrives
```

There are also `Monad` and `Monoid` implementations for `Future`
available from `cats.instances.future`:

```tut:book:silent
import cats.instances.future._
```

### Partitioning Sequences

We can partition a sequence
(actually anything that implements `Iterable`)
using the `grouped` method.
We'll use this to split off chunks of work for each CPU:

```tut:book
List(1, 2, 3, 4).grouped(2).toList
```

We can query the number of available CPUs on our machine
using this API call to the Java standard library:

```tut:book
Runtime.getRuntime.availableProcessors
```

### Parallel *foldMap*

Implement a parallel version of `foldMap` called `foldMapP`
using the tools described above:

```tut:book:silent
def foldMapP[A, B : Monoid]
    (values: Iterable[A])
    (func: A => B = (a: A) => a): Future[B] = ???
```

Start by splitting the input into a set of even chunks, one per CPU.
Create a future to do the work for each chunk using `Future.apply`,
and then `foldMap` cross the futures.

<div class="solution">
The annotated solution is below:

```tut:book:silent
import scala.concurrent.duration.Duration

def foldMapP[A, B: Monoid]
    (values: Iterable[A])
    (func: A => B = (a: A) => a): Future[B] = {
  // Calculate the number of items to pass to each CPU:
  val numCores: Int = Runtime.getRuntime.availableProcessors
  val groupSize: Int = (1.0 * values.size / numCores).ceil.toInt

  // Create one group for each CPU:
  val groups: Iterator[Iterable[A]] =
    values.grouped(groupSize)

  // Create a future to foldMap each group:
  val futures: Iterator[Future[B]] =
    groups map { group =>
      Future {
        group.foldLeft(Monoid[B].empty)(_ |+| func(_))
      }
    }

  // foldMap over the groups to calculate a final result:
  Future.sequence(futures) map { iterable =>
    iterable.foldLeft(Monoid[B].empty)(_ |+| _)
  }
}
```

```tut:book
Await.result(foldMapP(1 to 1000000)(), 1.second)
```
</div>

## Monadic *foldMap*

It's useful to allow the user of `foldMap`
to perform monadic actions within their mapping function.
This, for example, allows the mapping to indicate failure
by returning an `Option`.

Implement a variant of `foldMap` (without parallelism)
called `foldMapM` that allows this.
Here's the basic type signature---add
implicit parameters and context bounds as necessary to make your code compile:

```tut:book:silent
import scala.language.higherKinds

def foldMapM[A, M[_], B](iter: Iterable[A])(f: A => M[B]): M[B] =
  ???
```

```tut:book:invisible
import cats.Monad
import cats.syntax.applicative._
import cats.syntax.flatMap._
import cats.syntax.functor._

def foldMapM[A, M[_]: Monad, B: Monoid](iter: Iterable[A])(f: A => M[B]): M[B] =
  iter.foldLeft(Monoid[B].empty.pure[M]) { (accum, elt) =>
    for {
      a <- accum
      b <- f(elt)
    } yield a |+| b
  }
```

The focus here is on the monadic component
so base your code on `foldMap` for simplicity.
Here are some examples of use:

```tut:book:silent
import cats.instances.int._
import cats.instances.option._
import cats.instances.list._

val seq = List(1, 2, 3)
```

```tut:book
foldMapM(seq)(a => Option(a))

foldMapM(seq)(a => List(a))

foldMap(seq)(a => if(a % 2 == 0) Option(a) else Option.empty[Int])
```

<div class="solution">
First we change
the type of our `func` parameter
from `A => B` to `A => M[B]`
and bring in the `Monoid` for `M`.
Then we tweak the method implementation
to `flatMap` over the monad and call `|+|`:

```tut:book:silent
import cats.Monad
import cats.syntax.applicative._
import cats.syntax.flatMap._
import cats.syntax.functor._

def foldMapM[A, M[_]: Monad, B: Monoid](iter: Iterable[A])(f: A => M[B]): M[B] =
  iter.foldLeft(Monoid[B].empty.pure[M]) { (accum, elt) =>
    for {
      a <- accum
      b <- f(elt)
    } yield a |+| b
  }
```
</div>

### Exercise: Everything is Monadic

We can unify monadic and normal code by
using the `Id` monad described in the [Monads chapter](#id-monad).
Using this trick, implement a default `func` parameter for `foldMapM`.
 This allows us to write code like:

```scala
foldMapM(seq)
```

<div class="solution">
We have `Monad[B]` in scope in our method header,
so all we need to do is use the `point` syntax:

```tut:book:silent
import cats.Id
import cats.syntax.flatMap._
import cats.syntax.functor._

def foldMapM[A, M[_] : Monad, B: Monoid]
    (iter: Iterable[A])
    (f: A => M[B] = (a: A) => a.pure[Id]): M[B] =
  iter.foldLeft(Monoid[B].empty.pure[M]) { (accum, elt) =>
    for {
      a <- accum
      b <- f(elt)
    } yield a |+| b
  }
```
</div>

It also allows us to implement `foldMap` in terms of `foldMapM`. Try it!

<div class="solution">
```tut:book:silent
def foldMap[A, B : Monoid](iter: Iterable[A])(f: A => B = (a: A) => a): B =
  foldMapM[A, Id, B](iter) { a => f(a).pure[Id] }
```
</div>

### Exercise: Seeing is Believing

Call `foldMapM` using `Either`
and verify that it really does
stop execution as soon an error is encountered.
Start by writing a type alias to convert `Either`
to a type constructor with one parameter.
We'll use `Either.catchOnly` to read input,
so define your alias using an appropriate error type:

```tut:book:silent
import cats.instances.either._
import cats.syntax.either._
```

```tut:book
Either.catchOnly[NumberFormatException]("Cat".toInt)

Either.catchOnly[NumberFormatException]("1".toInt)
```

Once you have your type alias, call `foldMapM`.
Start with a sequence of `Strings`---both valid and invalid input---and
see what results you get:

<div class="solution">
The `catchOnly` approach gives us a `Either[NumberFormatException, Int]`
so we'll go with `NumberFormatException` as our error type:

```tut:book:silent
type ParseResult[A] = Either[NumberFormatException, A]
```

Now we can use `foldMapM`.
The resulting code iterates over the sequence,
adding up numbers using the `Monoid` for `Int`
until a `NumberFormatException` is encountered.
At that point the `Monad` for `Either` fails fast,
returning the failure without processing the rest of the list:

```tut:book
foldMapM[String, ParseResult, Int](List("1", "2", "3")) { str =>
  Either.catchOnly[NumberFormatException](str.toInt)
}

foldMapM[String, ParseResult, Int](List("1", "x", "3")) { str =>
  Either.catchOnly[NumberFormatException](str.toInt)
}
```
</div>

## Parallel Monadic *foldMap*

We've seen that we can extend `foldMap`
to work over a monad to add error handling (and other effects)
to the sequential version of `foldMap`.
What about a monadic parallel version of `foldMap`?

The straightforward extension of `foldMapP` doesn't really do what we want.
Because we're working in parallel
we can encounter multiple errors---one per thread---and
we would ideally report *all* of these errors.
Our current implementation will only report the first.

When we looked at applicatives in Chapter [@sec:applicatives]
we saw they allowed us to accumulate errors,
rather than stopping on the first error as inherently sequential monads do.
If we converted the monad to an applicative
we could use this accumulate all the errors we encounter.

It's easy enough to hard-code a choice of monad and applicative,
we can do a lot better than that.
If we allow the user to specify the transformation
they gain the flexibility to choose the error handling strategy
appropriate for their task.
They might want fail-fast error handling, accumulating all errors,
or even ignoring errors and replacing them with the identity.

This transformation should accept a `F[_]` and return a `G[_]`.
This concept known as a *natural transformation*[^kind-polymorphism].

[^kind-polymorphism]: Why can't we use a function with type `F => G` to do this?
The reason is the kinds are wrong.
`F` is a type with kind `*`,
while `F[_]` is a type constructor with kind `* => *`.
Scala provides no way to abstract over kinds.
Such a feature is known as *kind polymorphism*.

Implement `foldMapPM` with a user specified `NaturalTransformation`
to convert our `Monad` to an `Applicative`.
As every `Monad` is an `Applicative` define a default value
using the identity natural transformation.

## *foldMap* in the Real World

We've implemented a sequence processing abstraction, `foldMap`,
based on monoidal addition of elements.
We've seen that we can extend this to more flexible situations,
particularly user-specified error handling,
by combining monads, applicative functors, and natural transformations.
The result is a powerful and general framework
for sequential and parallel processing.
Is it useful, however?

It turns out that yes, this concept is very useful.
As we mentioned previously
the core idea of monoidal addition underlies [Summingbird][link-summingbird],
Twitter's framework that powers all their internal data processing jobs.

Monoids are not restricted to simple tasks like addition and string concatenation.
Most of the tasks data scientists perform in their day-to-day analyses can be cast as monoids.
There are monoids for all the following:

- approximate sets such as the Bloom filter;
- set cardinality estimators, such as the HyperLogLog algorithm;
- vectors and hence vector operations like stochastic gradient descent;
- quantile estimators such as the t-digest

to name but a few.
