# 4.2 Monad in Cats

## 4.2.1 The Monad Type Class

Cats 的 monad 定义在 `cats.Monad` 中，`Monad` 继承了 `FlatMap` 和 `Applicative`：

* 从 `FlatMap` 获得 `flatMap` 函数
* 从 `Applicative` 获得 `pure` 函数

因为 `Applicative` 继承 `Functor`，因此 `Monad` 简介获得了 `map` 函数。

下面是直接使用 `pure` `flatMap` 和 `map` 的简单例子：

```Scala
import cats.Monad
import cats.instances.list._
import cats.instances.option._

val xs = List(1, 2, 3)
val listMonad = Monad[List]

listMonad.flatMap(xs)(x ⇒ listMonad.pure(x * 2))  // 2, 4, 6
listMonad.map(xs)(_ + 1)  // 2, 3, 4

val o = Option(1)
val optionMonad = Monad[Option]

optionMonad.flatMap(o)(x ⇒ optionMonad.pure(x + 2))  // Some(3)
```

## 4.2.2 Default Instances

Cats 为 Scala 标准库中的所有 monad（例如 `List` `Option` 等）都提供了 instance：

```Scala
import cats.Monad
import cats.instances.list._
import cats.instances.option._
import cats.instances.vector._

Monad[Option].map(Some(1))(_ + 1)  // Some(2)
Monad[Option].flatMap(Some(1))(x ⇒ Some(x + 1))  // Some(2)
Monad[Option].pure(2)  // Some(2)

Monad[List].map(List(1, 2, 3))(_ + 1)  // 2, 3, 4
Monad[List].flatMap(List(1, 2, 3))(x ⇒ List(x, 2 * x))  // List(1, 2, 2, 4, 3, 6)


Monad[Vector].map(Vector(1, 2, 3))(_ + 1)  // Vector(2, 3, 4)
```

Cats 也为 `Future` 定义了 `Monad[Future]`，但与 `Future` 自身携带的 `map` 和 `flatMap` 方法不同，`Monad[Future]` 的 `flatMap` 不能接收 `ExecutionContext` 作为参数，因为 `Monad` 的定义中就没有，所以在执行 `Monad.apply(Future)`（即 `Monad[Future]`）时，要求 `impclicit` 作用域中存在隐式的 `ExecutionContext`：

```Scala
import cats.Monad
import cats.instances.future._

import scala.concurrent.{Await, Future}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._

val fm: Monad[Future] = Monad[Future]

val future = fm.flatMap(fm.pure(1))(x ⇒ fm.pure(x * 2))

Await.result(future, 1.second)  // 2
```

## 4.2.3 Monad Syntax

monad 的 syntax 分布在 3 个包中：

1. `cats.syntax.flatMap`：提供 `flatMap` 语法
2. `cats.syntax.applicative`：提供 `pure` 语法
3. `cats.syntax.functor`：提供 `map` 语法

可以通过 `cats.implicits` 导入所有，也可以显式按需导入。

可以通过 `pure` 函数创建 monad 实例，但注意，需要指定类型参数，否则会导致歧义：

```Scala
import cats.Monad
import cats.instances.list._
import cats.instances.option._
import cats.syntax.applicative._

1.pure[List]  // List(1)
1.pure[Option]  // Some(1)
```

如果不指定类型参数，如 `1.pure`，则 Scala 编译器无法得知，我们到底是想把 `1` 转换为 `List` 还是 `Option`，因为两者都合法的。

因此 `List` 和 `Option` 都自定了自己的 `map` 和 `flatMap` 函数，因此无法直接用他们展示 `Monad[List]` 实例中 `map` 和 `flatMap` 函数的用法。

现在实现一个函数，该函数的参数可以接受所有 monadic context（例如 `List`, `Option`），以此来展示 `Monad.map` 和 `Monad.flatMap` 的用法：

```Scala
import cats.Monad
import cats.syntax.flatMap._
import cats.syntax.functor._

import scala.language.higherKinds

def sumSquare[F[_]: Monad](x: F[Int], y: F[Int]): F[Int] =
  x.flatMap(a ⇒ y.map(b ⇒ a * a + b * b))
```

`sumSquare` 接受两个 `F[Int]` 作为参数，并对 monad context 求平方和，任何 `F[Int]` 都可以作为参数传递给 `sumSquare`，例如 `Vector` `List` `Option` `Future` 等，前提是作用域中有 **对应的 Monad[F] 实例**：

```Scala
import cats.instances.list._

sumSquare2(List(1, 2), List(3, 4, 5))  // List(10, 17, 26, 13, 20, 29)
sumSquare2(List(1, 2), List.empty[Int])  // Nil

import cats.instances.option._

sumSquare(Option(1), Option(2))  // Some(5)
sumSquare(Option(1), None: Option[Int])  // None
```

可以用 for 解析重构 `sumSquare`：

```Scala
def sumSquare2[F[_]: Monad](x: F[Int], y: F[Int]): F[Int] =
  for {
    a ← x
    b ← y
  } yield a * a + b * b
```
