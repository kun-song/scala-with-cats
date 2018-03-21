# 6.3 Semigroupal Applied to Different Types

有时 `Semigroupal` 实例的行为与预期相差很远，6.2 展示了 `Semigroupal[Option]` 实例的行为，本节展示其他实例。

## `Future`

`Semigroupal[Future]` 语义为并行计算（实际并非并行，也是串行），而非串行计算：

```Scala
import cats.Semigroupal
import cats.instances.future._

import scala.concurrent.{Await, Future}
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global

val a = Semigroupal[Future].product(Future(111), Future(6))

// (Int, Int) = (111,6)
Await.result(a, 1.second)
```
* `Future(111)` 和 `Future(6)` 在创建后立即执行，在执行 `product` 时，它们已经完成

也可以使用 `cats.syntax.apply`：

```Scala
import cats.instances.future._
import cats.syntax.apply._

import scala.concurrent.{Await, Future}
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global

case class Cat(name: String, born: Int, color: String)

val catFuture = (
  Future("Mike"),
  Future(12),
  Future("yellow")
).mapN(Cat.apply)

// Cat = Cat(Mike,12,yellow)
Await.result(catFuture, 1.second)
```

## `List`

`Semigroupal[List]` 实例行为有点蛋疼，它并不是 zip the lists，而是求各个元素的 **笛卡尔积**：

```Scala
import cats.Semigroupal
import cats.instances.list._

// List[(Int, Int)] = List((1,3), (1,4), (2,3), (2,4))
val x = Semigroupal[List].product(List(1, 2), List(3, 4))
```

## `Either`

本章开头讨论了两种错误处理策略：

* fail-fast error-handling
* accumulating error-handling

因此我们可能会认为 `Semigroupal[Either]` 实现了 accumulating error-handling，但实际并非如此：

```Scala
import cats.Semigroupal
import cats.instances.either._

type ErrorOr[A] = Either[Vector[String], A]

// ErrorOr[(Nothing, Nothing)] = Left(Vector(error 1))
val x = Semigroupal[ErrorOr].product(
  Left(Vector("error 1")),
  Left(Vector("error 2"))
)
```
* `product` 遇到第一个参数是 `Left` 就停止了，fail-fast
* `product` 行为与 `flatMap` 一样，都是 fail-fast

## 6.3.1 Semigroupal Applied to Monads

`Semigroupal[Either]` 和 `Semigroupal[List]` 行为怪异的原因是：`Either` 和 `List` 都是 Monad！！！

因为 `Monad` 继承了 `Semigroupal`，为了保持 **一致的语义**，Cats 中的 Monad 都是以 `map` + `flatMap` 实现的，因此对于 `Semogroupal[Monad]` 而言，`product` 语义与 `flatMap` 是一致的。

这种一致性看似会导致某些无用的行为（`Either` `List`），但实际上，一致性对于更高层次的抽象而言，非常有用。

那为什么前面说 `Semigroupal[Future].product` 是并行的呢？

这是因为在执行 `product` 之前，`Future` 早已开始执行，实际 `product` 与 `flatMap` 语义相同，都是串行执行，**create-then-flatMap** 模式可以做到与 `product` 完全相同的并行效果：

```Scala
val a = Future("Future 1")
val b = Future("Future 2")

for {
  x <- a
  y <- b
} yield (x, y)
```

### 6.3.1.1 Exercise: The Product of Monads

使用 `flatMap` + `map` 实现 `product` 函数：

```Scala
import cats.Monad
import cats.syntax.functor._  // for map
import cats.syntax.flatMap._  // for flatMap

import scala.language.higherKinds

def product1[M[_]: Monad, A, B](ma: M[A], mb: M[B]): M[(A, B)] =
  Monad[M].flatMap(ma)(a ⇒ Monad[M].map(mb)(b ⇒ (a, b)))

def product2[M[_]: Monad, A, B](ma: M[A], mb: M[B]): M[(A, B)] =
  ma.flatMap(a ⇒ mb.map(b ⇒ (a, b)))

def product[M[_]: Monad, A, B](ma: M[A], mb: M[B]): M[(A, B)] =
  for {
    a ← ma
    b ← mb
  } yield (a, b)
```