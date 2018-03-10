# 4.3 The Identity Monad

4.2 节为了展示 `map` 和 `flatMap` 实现了一个函数：

```Scala
import cats.Monad
import cats.syntax.flatMap._
import cats.syntax.functor._

import scala.language.higherKinds

def sumSquare2[F[_]: Monad](x: F[Int], y: F[Int]): F[Int] =
  for {
    a ← x
    b ← y
  } yield a * a + b * b
```

`sumSquare` 非常非常通用，可用于任何具备 `Monad[F]` 实例的 `F[Int]` 类型，比如 `List[Int]` `Option[Int]` 等，这已经非常通用了，如果没有 monad，我们只能为 `List[Int]` 和 `Option[Int]` 分别实现 `sumSquare` 函数。

美中不足的是，无法使用 plain value 调用 `sumSquare`，例如 `sumSquare(1, 2)` 会报错！

如果 `sumSquare` 既能用于 monad，也能用于非 monad，那 `sumSquare` 就无敌通用了！

幸运的是 Cats 通过 `Id` 弥补了 monad 和非 monad 之间鸿沟，借助 `Id`，可以用 plain value 调用 `sumSquare` 了：

```Scala
// use Id type class to make sumSquare work with non-monad
import cats.Id

sumSquare(1: Id[Int], 2: Id[Int])
```

上面例子中，我们先将 `Int` 转换为 `Id[Int]`，然后调用 `sumSquare` 函数，最后返回值类型也是 `Id[Int]`。

要搞清楚中间发生了什么，需要看下 `Id` 的定义：

```Scala
package cats

type Id[A] = A
```

可以看到，`Id` 就是一个类型别名，作用是将 atomic type **转换** 为 **single-parameter type constructor**，可以将 **任何** 值转换为对应的 `Id` 实例：

```Scala
import cats.Id

// Id[Int]
1: Id[Int]

// Id[String]
"Hello": Id[String]

// Id[List[Int]]
List(1, 2, 3): Id[List[Int]]
```

Cats 为 `Id` 实现了很多 type class instances，例如 `Functor[Id]` `Monad[Id]` 等：

```Scala
import cats.Monad
import cats.Id

val idMonad: Monad[Id] = Monad[Id]

val a: Id[Int] = idMonad.pure(111)
val b: Id[Int] = idMonad.flatMap(a)(x ⇒ idMonad.pure(x * 5))
// or val b: Id[Int] = idMonad.flatMap(a)(_ * 5)

import cats.syntax.flatMap._
import cats.syntax.functor._

for {
  x ← a
  y ← b
} yield x + y  // 666 Id[Int]
```

**注意**：

能实现可同时用于 monad 和 non-monad 的代码非常有用，例如可以在生产环境用 `Monad[Future]` 实例调用，而在测试环境使用 `Monad[Id]` 调用。

## 4.3.1 Exercise: Monadic Secret Identities

为 `Id` 实现 `pure` `map` 和 `flatMap` 函数：

```Scala
def pure[A](value: A): Id[A] = value

def flatMap[A, B](ia: Id[A])(f: A ⇒ Id[B]): Id[B] = f(ia)

def map[A, B](ia: Id[A])(f: A ⇒ B): Id[B] = f(ia)
```

因为 `A` 和 `Id[A]` 实际是相同类型，所以实现起来非常简单。

>`map` 和 `flatMap` 都隐藏了 **某种复杂性**
>
>1. 前面说过，每种 `Monad[F]` 实例都为用户隐藏了某种复杂性，如 `Monad[Option]` 隐藏了可能没有返回值，而对于 `Monad[Id`，**没有任何需要隐藏的复杂性**，这是 monad 的极端简化场景；
>1. 因此，`map` 和 `flatMap` 的实现 **完全一致**，符合预期；
