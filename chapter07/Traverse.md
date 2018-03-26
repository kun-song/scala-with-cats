# 7.2 Traverse

`foldLeft` 和 `foldRight` 固然是很强大的遍历函数，但因为需要使用者提供 accumulator + binary function，所以相对繁琐，`Traverse` 是比 `Foldable` 更加抽象的 type class，提供了更加方便的遍历方式。

## 7.2.1 Traversing with Futures

可通过 Scala 标准库中的 `Future.traverse` 和 `Future.sequence` 来介绍 `Traverse`，这两个函数是特定于 `Future` 的 traverse pattern 实现。

假设有一个 hostname 列表，和一个可查询指定服务器运行时间的函数：

```Scala
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

val hostnames = List(
  "alpha.example.com",
  "beta.example.com",
  "gamma.demo.com"
)

def getUptime(hostname: String): Future[Int] =
  Future(hostname.length * 60)
```

如果要查询所有服务器的运行时间，第一时间想到的一般是 `hostnames.map(getUptime)`，但其返回类型为 `List[Future[Int]]`，我们想要的是 `Future[List[Int]]`，可以用 `foldLeft` 实现：

```Scala
val allUptimes: Future[List[Int]] =
  hostnames.foldLeft(Future(List.empty[Int])) { (accf, name) ⇒
    for {
      time ← getUptime(name)
      acc  ← accf
    } yield acc :+ time
  }

// List[Int] = List(1020, 960, 840)
Await.result(allUptimes, 1.second)
```

上面例子中的套路 **非常常见**，如果使用 `foldLeft`，则每次都需要 **重复** 相同的代码片段，`Future.traverse` 就是 **专门** 为该场景设计的：

```Scala
val allUptimes: Future[List[Int]] =
  Future.traverse(hostnames)(getUptime)

// List[Int] = List(1020, 960, 840)
Await.result(allUptimes, 1.second)
```

`Future.traverse` 实现如下，与上面 `foldLeft` 的例子非常相像：

```Scala
def traverse[A, B](values: List[A])(func: A => Future[B]): Future[List[B]] =
  values.foldLeft(Future(List.empty[A])) { (accum, host) =>
    val item = func(host)
    for {
      accum <- accum
      item  <- item
    } yield accum :+ item
  }
```

`Future.traverse` 对 `foldLeft` 进一步抽象，实现了如下逻辑：

1. 给定 `List[A]` 和函数 `A => Future[B]`
2. 获得 `Future[List[A]]`

还有一个类似函数，`Future.sequence`：

1. 给定 `List[Future[A]]`
2. 获得 `Future[List[A]]`

`sequence` 是 `traverse` 的特殊场景：

```Scala
object Future {
  def sequence[B](futures: List[Future[B]]): Future[List[B]] =
    traverse(futures)(identity)
}
```

`Future.traverse/sequence` 不仅能用于 `List`，实际上他们可用于 **所有** Scala 标准集合。

`traverse` 和 `sequence` 如此有用，如果不仅 `Future` 能用，其他类型也能用就好了，为达到该目的，Cats 的 `Traverse` type class 对该模式进行抽象，使其可用于 **所有 `Applicative`，所以 `List` `Option` 也有自己的 `traverse` 和 `sequence` 函数啦！

## 7.2.2 Traversing with Applicatives

实际上，可以用 `Applicative` 实现 `traverse` 函数，使用 `foldLeft` 时的 accumulator 是 `Future(List.empty[Int])`，这可以用 `Applicative.pure` 替代：

```Scala
import cats.instances.future._   // for Applicative[Future] instance
import cats.syntax.applicative._ // for pure

List.empty[Int].pure[Future]
```

`foldLeft` 中的 binary function 为：

```Scala
def combine1(accm: Future[List[Int]], host: String): Future[List[Int]]
  (accum, host) =>
    val item = func(host)
    for {
      accum <- accum
      item  <- item
    } yield accum :+ item
```

`combine1` 实际与 `Semigroupal.combine` 等价：

```Scala
def combine2(acc: Future[List[Int]], hostname: String): Future[List[Int]] =
  (acc, getUptime(hostname)).mapN(_ :+ _)
```

分别将 accumulator 和 binary function 用以上代码替代，可以重新实现 `traverse`：

```Scala
import scala.language.higherKinds

import cats.Applicative
import cats.syntax.apply._
import cats.syntax.applicative._

def listTraverse[F[_]: Applicative, A, B](xs: List[A])(f: A ⇒ F[B]): F[List[B]]=
  xs.foldLeft(List.empty[B].pure[F]) { (accF, a) ⇒
    (accF, f(a)).mapN(_ :+ _)
  }

def listSequence[F[_]: Applicative, A](xs: List[F[A]]): F[List[A]] =
  listTraverse(xs)(identity)
```

可以用 `listSequence` 替代 `Future.traverse`：

```Scala
val allUptimes: Future[List[Int]] = listTraverse(hostnames)(getUptime)

// List[Int] = List(1020, 960, 840)
Await.result(allUptimes, 1.second)
```

而且 `listTraverse` 可用于任意 `Applicative`，而非仅仅 `Future`。

### 7.2.2.1 Exercise: Traversing with Vectors

`listTraverse` 可以用于 `Vector`：

```Scala
import cats.instances.vector._ // for Applicative[Vector]

// Vector[List[Int]] = Vector(List(1, 3), List(1, 4), List(2, 3), List(2, 4))
listSequence(List(Vector(1, 2), Vector(3, 4)))

// Vector[List[Int]] = Vector(List(1, 3, 5), List(1, 3, 6), List(1, 4, 5), List(1, 4, 6), List(2, 3, 5), List(2, 3, 6), List(2, 4, 5), List(2, 4, 6))
listSequence(List(Vector(1, 2), Vector(3, 4), Vector(5, 6)))
```

输入类型为 `List[Vector[Int]]`，因此 `F` 为 `Vector`，使用 `Applicative[Vector].product` 进行组合，但因为 `Vector` 是 `Monad`，所以其 `product` 是基于 `flatMap` 实现的，因此会求各个 `Vector` 元素的 **笛卡尔积**。 

### 7.2.2.2 Exercise: Traversing with Options

`Option` 也是 `Applicative`，可用于 `listTraverse`：

```Scala
import cats.instances.option._

def process(input: List[Int]): Option[List[Int]] =
  listTraverse(input)(x ⇒ if (x % 2 == 0) Some(x) else None)
```

输入为 `List[Int]`，函数为 `Int => Option[Int]`，所以结果为 `Option[List[Int]]`，底层使用 `Applicative[Option].product` 执行计算，因为 `Option` 是 `Monad`，因此 `product` 是以 `flatMap` 实现的，是 fail-fast 的，只要有一个 `None`，则结果为 `None`：

```Scala
// Option[List[Int]] = None
process(List(1, 2, 3))

// Option[List[Int]] = Some(List(2, 4, 6))
process(List(2, 4, 6))
```

### 7.2.2.3 Exercise: Traversing with Validated

使用 `Validated` 替代 `Option` 重新实现 `process`：

```Scala
import cats.data.Validated
import cats.instances.list._
import cats.syntax.validated._

type ErrorOr[A] = Validated[List[String], A]

def process(input: List[Int]): ErrorOr[List[Int]] =
  listTraverse(input) { x ⇒
    if (x % 2 == 0) x.valid
    else List(s"$x is not event").invalid[Int]
  }
```

返回类型为 `ErrorOr[List[Int]]`，进而是 `Validated[List[String], List[Int]]`，因为 `Validated` 是 accumulating error-handling，因此返回值要么是 `List[Int]`，要么是包含所有错误信息的 `List[String]`：

```Scala
// ErrorOr[List[Int]] = Invalid(List(1 is not event, 3 is not event))
process(List(1, 2, 3))

// ErrorOr[List[Int]] = Valid(List(2, 4, 6))
process(List(2, 4, 6))
```

## 7.2.3 Traverse in Cats

`listTraverse` 和 `listSequence` 可以用于任意 `Applicative`，但序列类型限制在 `List`，Cats 的 `Traverse` 更加通用，可以用于：

* 任意 sequence
* 任意 `Applicative`

其简略定义如下：

```Scala
package cats

trait Traverse[F[_]] {
  def traverse[G[_]: Applicative, A, B](inputs: F[A])(func: A => G[B]): G[F[B]]

  def sequence[G[_]: Applicative, B](inputs: F[G[B]]): G[F[B]] =
    traverse(inputs)(identity)
}
```

Cats 为 `List` `Vector` `Stream` `Option` `Either` 以及很多其他类型都实现了 `Traverse` 实例，可以用 `Traverse.apply` 获取合适的实例使用：

```Scala
val allUptimes: Future[List[Int]] = Traverse[List].traverse(hostnames)(getUptime)

// List[Int] = List(1020, 960, 840)
Await.result(allUptimes, 1.second)

val numbers: List[Future[Int]] = List(Future(1), Future(2), Future(3))
val numbers2: Future[List[Int]] = Traverse[List].sequence(numbers)

// List[Int] = List(1, 2, 3)
Await.result(numbers2, 1.second)
```

通过 `cats.syntax.traverse` 可以简化上述例子：

```Scala
import cats.syntax.traverse._

val allUptimes: Future[List[Int]] = hostnames.traverse(getUptime)
val numbers2: Future[List[Int]] = numbers.sequence
```

使用 `traverse` 和 `sequence` 明显比 `foldLeft` 更加紧凑、可读。