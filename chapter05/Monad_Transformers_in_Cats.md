# 5.3 Monad Transformers in Cats

每个 monad transformer 都是一个数据类型，定义在 `cats.data` 中，借助 monad transformer 可以将 **嵌套** 的 monad 变成单个 monad。

要使用 monad transformer，必须理解：

1. the available transformer classes;
2. how to build **stacks of monads** using transformers;
3. how to construct instances of a **monad stack**; and
4. how to pull apart a stack to access the wrapped monads.

## 5.3.1 The Monad Transformer Classes

在 Cats 中，对 monad `Foo`，通常对应有 monad transformer `FooT`。实际上，Cats 中很多 Monad 其实是通过组合 monad transformer 和 `Id` monad 生成的。

一些常见的 monad 和其 monad transformer 如下：

* `cats.data.OptionT` for `Option`;
* `cats.data.EitherT` for `Either`;
* `cats.data.ReaderT` for `Reader`;
* `cats.data.WriterT` for `Writer`;
* `cats.data.StateT` for `State`;
* `cats.data.IdT` for the `Id` monad.

>Kleisli Arrows
>
>In Section 4.8 we mentioned that the `Reader` monad was a specialisation of a more general concept called a “kleisli arrow”, represented in Cats as `cats.data.Kleisli`.
>
>We can now reveal that `Kleisli` and `ReaderT` are, in fact, the same thing! `ReaderT` is actually a type alias for `Kleisli`. Hence why we were creating `Reader`s last chapter and seeing `Kleisli`s on the console.

## 5.3.2 Building Monad Stacks

所有 monad transformer 遵循 **相同套路**：

1. transformer 本身： monad stack 中的 **inner monad**
2. transformer 第一个类型参数：代表 monad stack 中的 **outer monad**
3. transformer 第二个类型参数：最深层的 monad context 类型

例如前面例子中的 `ListOption`，它是 `OptionT[List, A]` 的别名，而 `OptionT[List, A]` 实际上是 `List[Option[A]]`，即创建 monad stack 时，采用 **由内而外** 的顺序构建：

```Scala
type ListOption[A] = OptionT[List, A]
```

**很多 Monad + 全部 Monad Transformer** 有 **两个** 以上的类型参数，此时需要为中间步骤定义类型别名。

假如想用 `Either` 包裹 `Option`：

* 因为 `Option` 是 **inner monad**，因此使用 `OptionT` transformer
* 因为 `Either` 是 **outer monad**，因此 `OptionT` 的第一个类型参数为 `Either`

按照上面的推理，最终组合出来的 monad 为：

```Scala
type EitherOption[A] = OptionT[Either[?, ?], A]
```

`Either` 是具有两个类型参数的 **type constructor**，而根据 monad 定义，`EitherOption` 只有一个类型参数，两者不匹配！

解决方式就是 **type alias**，需要固定 `Either` 其中一个类型，将其转换为 **只有一个 type parameter 的 type constructor**，这里我们将 `Either` 左侧的类型固定为 `String`：

```Scala
// Alias Either to a type constructor with one parameter
type ErrorOr[A] = Either[String, A]
```

固定 `Either` 左侧为 `String`，并定义类型别名后，`ErrorOr` 变成了只有一个类型参数的 type constructor，可以用于 monad transformer 了：

```Scala
type ErrorOrOption[A] = OptionT[ErrorOr, A]
```
* `OptionT[ErrorOr, A]`，`OptionT` 为 **inner monad**，因此 `Option` 是 `ErrorOr` 定义中的 `A`，即 `Either[String, Option]`

使用 monad transformer `OptionT` 进行组合后，`ErrorOrOption` 是一个新 monad，可使用 `pure` `map` 和 `flatMap` 等 monad 函数：

```Scala
import cats.data.OptionT
import cats.instances.either._
import cats.syntax.applicative._

type ErrorOr[A] = Either[String, A]

type ErrorOrOption[A] = OptionT[ErrorOr, A]

// a: ErrorOrOption[Int] = OptionT(Right(Some(111)))
val a = 111.pure[ErrorOrOption]

// b: ErrorOrOption[Int] = OptionT(Right(Some(6)))
val b = 6.pure[ErrorOrOption]

// res0: cats.data.OptionT[ErrorOr,Int] = OptionT(Right(Some(666)))
a.flatMap { x ⇒
  b.map { y ⇒
    x * y
  }
}
```

当需要嵌套 **3 层及以上** monad 时，更需要 **type alias**。

假如想创建 **a `Future` of `Either` of `Option`**，按照 **从内向外** 的原则，需要创建的 monad 为 **a `OptionT` of `EitherT` of `Future`**，直接按照定义写：

```Scala
type FutureEitherOption[A] = OptionT[EitherT[?, ?, A], A]
```

`EiterT` 需要 3 个类型参数，而 `FutureEitherOption` 只提供了一个类型参数 `A`，那 `EitherT` 另外两个参数应该是什么呢？尝试直接将 `F` 固定为 `Future`，将 `E` 固定为 `String` 试试：

```Scala
type FutureEitherOption[A] = OptionT[EitherT[Future, String, A], A]
```

编译后发现，该定义会报如下错误：

```Scala
Error:(8, 39) cats.data.EitherT[scala.concurrent.Future,String,A] takes no type parameters, expected: one
type FutureEitherOption[A] = OptionT[EitherT[Future, String, A], A]
                                     ^
```

错误提示为 `EitherT` **takes no type parameters**，而 Scala 要求 `OptionT` 第一个参数必须 **有且仅有** 一个类型参数，因此在 1 行内根本无法完成定义。

`EitherT` 简单定义如下：

```Scala
case class EitherT[F[_], E, A](stack: F[Either[E, A]]) {
  // etc...
}
```
* `F[_]` 是 **outer monad**
* `E` 是错误类型
* `A` 是正确类型

为了将 `EitherT` 转换为 **只有 1 个类型参数** 的 type constructor，此处，需要使用 **类型别名**，还是将 `F` 固定为 `Future`，将 `E` 固定为 `String`:

```Scala
type FutureEither[A] = EitherT[Future, String, A]
```

`FutureEither` 变成了只有一个类型参数的 type constructor，可以用作 `OptionT` 的第一个参数：

```Scala
type FutureEitherOption[A] = OptionT[FutureEither, A]
```

`FutureEitherOption` 是我们组合后的 monad，它由 3 个 monad 嵌套组成，使用如下：

```Scala
import cats.data.{EitherT, OptionT}
import cats.instances.future._
import cats.syntax.applicative._

import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

type FutureEither[A] = EitherT[Future, String, A]

type FutureEitherOption[A] = OptionT[FutureEither, A]


for {
  a ← 111.pure[FutureEitherOption]
  b ← 6.pure[FutureEitherOption]
} yield a * b
```

`FutureEitherOption` 的 `map` 和 `flatMap` 将横切 3 个 monad，使用很方便。

## 5.3.3 Constructing and Unpacking Instances

可以用 monad transfromer 的 `apply` 或者 `Applicative` 的 `pure` 函数来创建 monad stacks 的实例：

```Scala
type ErrorOr[A] = Either[String, A]
type ErrorOrOption[A] = OptionT[ErrorOr, A]

// cats.data.OptionT[ErrorOr,Int] = OptionT(Right(Some(111))
val a = OptionT[ErrorOr, Int](Right(Some(111)))

// ErrorOrOption[Int] = OptionT(Right(Some(111)))
val b = 6.pure[ErrorOrOption]
```

实例创建完成后，可以 `value` 函数对其 **解包**，解包后获得内层的 monad，从而可使用普通 monad 的各种函数：

```Scala
// x: Option[Int] = Some(111)
val x = a.value.getOrElse(None)

// y: ErrorOr[Option[Int]] = Right(Some(6))
val y = b.value
```
* `a` 类型为 `OptionT[ErrorOr, Int]`，使用 `a.value` 获取 `OptionT` 内包裹的类型，即 `ErrorOr`，而 `ErrorOr` 实际包裹了 `Option`，所以最后获取的是 `ErrorOr[Option]`

每次调用 `value` 都会对 monad stack 执行一层解包，多是嵌套多层，则需要调用多次 `value`：

```Scala
import cats.data.{EitherT, OptionT}
import cats.instances.future._
import cats.instances.either._
import cats.syntax.applicative._

import scala.concurrent.{Await, Future}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._

type FutureEither[A] = EitherT[Future, String, A]

type FutureEitherOption[A] = OptionT[FutureEither, A]

val f = 111.pure[FutureEitherOption]
// e: scala.concurrent.Future[Either[String,Option[Int]]] = Future(Success(Right(Some(111))))
val e = f.value.value

val result = Await.result(e, 1.second)

val c =
  for {
    a ← 111.pure[FutureEitherOption]
    b ← 6.pure[FutureEitherOption]
  } yield a * b

Await.result(c.value.value, 1.second)
```

* `FutureEitherOption` 由两个 monad 嵌套组成，因此需要调用两个 `value`
* 调用两个 `value` 结果类型为 `Future(Success(Right(Some(111))))`
* 只有使用 `value` 解包后，才能用其调用 `Await.result` 函数

## 5.3.4 Default Instances

Cats 中很多 monad 是通过 **monad transformer + `Id` monad** 定义的，这保证了 monad 及其对应的 monad transformer 的 **API 一致性**。

`Reader` `Writer` 和 `State` 都是通过 transformer + `Id` 定义的：

```Scala
type Reader[E, A] = ReaderT[Id, E, A] // = Kleisli[Id, E, A]
type Writer[W, A] = WriterT[Id, W, A]
type State[S, A]  = StateT[Id, S, A]
```

有的 monad 与其 transformer 是分开定义的，这时，transformer 会尽量与其 monad 保持 API 一致，例如 `OptionT` 和 `EitherT`。

## 5.3.5 Usage Patterns

因为 monad transformer 以 **预定义好** 的方式组合使用 monad，因此很难 **大规模** 使用。若设计不好，则在应用各部分，可能需要不断的进行 unpack/repack，以实现对最内层的 content 进行操作。

有两种设计模式可以处理该问题。

### A single "Super Stack"

若代码天然具备内在统一性，则可以创建一个 super stack，应用中全部使用它。

例如在 web 应用中，可以决定每个 request handler 都是的，且都可能失败，其失败错误码都是 HTTP 标准错误，该场景中，可以组合 `Future` 和 `Either` 为新 monad，并在应用中广泛使用：

```Scala
sealed abstract class HttpError

final case class NotFound(item: String) extends HttpError
final case class BadRequest(msg: String) extends HttpError
// etc...

type FutureEither[A] = EitherT[Future, HttpError, A]
```

### Monad Transformers as Local "Glue Code" 

若代码非常复杂，无法保持一致性，在不同场景中需要使用不同的 **monad stack**，则可以仅将 monad stack 作为内部使用，即：

1. 模块之间暴露 untransformed stacks
2. 模块内部将其他模块的 untransformed stacks 进行转换，生成 monad stack，供内部使用

这样一来，每个模块内部都可以根据实际业务需要，挑选合适的 monad transformer 使用：

```Scala
import cats.data.Writer

type Logged[A] = Writer[List[String], A]

// Methods generally return untransformed stacks:
def parseNumber(str: String): Logged[Option[Int]] =
  util.Try(str.toInt).toOption match {
    case Some(num) => Writer(List(s"Read $str"), Some(num))
    case None      => Writer(List(s"Failed on $str"), None)
  }

// Consumers use monad transformers locally to simplify composition:
def addAll(a: String, b: String, c: String): Logged[Option[Int]] = {
  import cats.data.OptionT

  val result = for {
    a <- OptionT(parseNumber(a))
    b <- OptionT(parseNumber(b))
    c <- OptionT(parseNumber(c))
  } yield a + b + c

  result.value
}

// This approach doesn't force OptionT on other users' code:
val result1 = addAll("1", "2", "3")
// result1: Logged[Option[Int]] = WriterT((List(Read 1, Read 2, Read 3),Some(6)))

val result2 = addAll("1", "a", "3")
// result2: Logged[Option[Int]] = WriterT((List(Read 1, Failed on a),None))
```

**注意**：monad transformer 不是万能良药，适合不适合用取决于很多因素，例如团队的大小和经验，代码的复杂度等等。
