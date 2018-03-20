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
