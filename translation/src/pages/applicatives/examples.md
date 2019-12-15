## Semigroupal Applied to Different Types

`Semigroupal` doesn't always provide the behaviour we expect,
particularly for types that also have instances of `Monad`.
We have seen the behaviour of the `Semigroupal` for `Option`.
Let's look at some examples for other types.

**Future**

The semantics for `Future`
provide parallel as opposed to sequential execution:

```tut:book:silent
import cats.Semigroupal
import cats.instances.future._ // for Semigroupal
import scala.concurrent._
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global
import scala.language.higherKinds

val futurePair = Semigroupal[Future].
  product(Future("Hello"), Future(123))
```

```tut:book
Await.result(futurePair, 1.second)
```

The two `Futures` start executing the moment we create them,
so they are already calculating results
by the time we call `product`.
We can use apply syntax to zip fixed numbers of `Futures`:

```tut:book:silent
import cats.syntax.apply._ // for mapN

case class Cat(
  name: String,
  yearOfBirth: Int,
  favoriteFoods: List[String]
)

val futureCat = (
  Future("Garfield"),
  Future(1978),
  Future(List("Lasagne"))
).mapN(Cat.apply)
```

```tut:book
Await.result(futureCat, 1.second)
```

**List**

Combining `Lists` with `Semigroupal`
produces some potentially unexpected results.
We might expect code like the following to *zip* the lists,
but we actually get the cartesian product of their elements:

```tut:book:silent
import cats.Semigroupal
import cats.instances.list._ // for Semigroupal
```

```tut:book
Semigroupal[List].product(List(1, 2), List(3, 4))
```

This is perhaps surprising.
Zipping lists tends to be a more common operation.
We'll see why we get this behaviour in a moment.

**Either**

We opened this chapter with a discussion of
fail-fast versus accumulating error-handling.
We might expect `product` applied to `Either`
to accumulate errors instead of fail fast.
Again, perhaps surprisingly,
we find that `product` implements
the same fail-fast behaviour as `flatMap`:

```tut:book:silent
import cats.instances.either._ // for Semigroupal

type ErrorOr[A] = Either[Vector[String], A]
```

```tut:book
Semigroupal[ErrorOr].product(
  Left(Vector("Error 1")),
  Left(Vector("Error 2"))
)
```

In this example `product` sees the first failure and stops,
even though it is possible to examine the second parameter
and see that it is also a failure.

### Semigroupal Applied to Monads

The reason for the surprising results
for `List` and `Either` is that they are both monads.
To ensure consistent semantics,
Cats' `Monad` (which extends `Semigroupal`)
provides a standard definition of `product`
in terms of `map` and `flatMap`.
This gives what we might think of as
unexpected and less useful behaviour for a number of data types.
The consistency of semantics is important
for higher level abstractions,
but we don't know about those yet.

Even our results for `Future` are a trick of the light.
`flatMap` provides sequential ordering,
so `product` provides the same.
The parallel execution we observe
occurs because our constituent `Futures`
start running before we call `product`.
This is equivalent to the classic
create-then-flatMap pattern:

```tut:book:silent
val a = Future("Future 1")
val b = Future("Future 2")

for {
  x <- a
  y <- b
} yield (x, y)
```

So why bother with `Semigroupal` at all?
The answer is that we can create useful data types that
have instances of `Semigroupal` (and `Applicative`) but not `Monad`.
This frees us to implement `product` in different ways.
We'll examine this further in a moment
when we look at an alternative data type for error handling.

#### Exercise: The Product of Monads

Implement `product` in terms of `flatMap`:

```tut:book:silent
import cats.Monad

def product[M[_]: Monad, A, B](x: M[A], y: M[B]): M[(A, B)] =
  ???
```

<div class="solution">
We can implement `product`
in terms of `map` and `flatMap` like so:

```tut:book:silent
import cats.syntax.flatMap._ // for flatMap
import cats.syntax.functor._ // for map

def product[M[_]: Monad, A, B](x: M[A], y: M[B]): M[(A, B)] =
  x.flatMap(a => y.map(b => (a, b)))
```

Unsurprisingly, this code is equivalent to a for comprehension:

```tut:book:silent
def product[M[_]: Monad, A, B](x: M[A], y: M[B]): M[(A, B)] =
  for {
    a <- x
    b <- y
  } yield (a, b)
```

The semantics of `flatMap` are what give rise
to the behaviour for `List` and `Either`:

```tut:book:silent
import cats.instances.list._ // for Semigroupal
```

```tut:book
product(List(1, 2), List(3, 4))
```

```tut:book:silent
type ErrorOr[A] = Either[Vector[String], A]
```

```tut:book
product[ErrorOr, Int, Int](
  Left(Vector("Error 1")),
  Left(Vector("Error 2"))
)
```

</div>
