# 10.5 Kleislis

虽然已经实现了 `Check`，我们写了很多代码，但实现的功能却很少，其中 `Predicate` 本质就是函数 `A => Validated[E, A]`，而 `Check` 又仅仅是 `Predicate` 的 wrapper。

可以将 `A => Validated[E, A]` 抽象为 `A => F[B]`，这是传递给 `flatMap` 的函数，假设有如下操作：

1. 使用函数 `A => F[B]`（例如 `Pure`）将值 lift 到 Monad context 中；
2. 使用 `flatMap` 在该 Monad 上施加很多数据转换；

操作如下图：

![img](../images/sequencing-monadic-transforms.svg)

若给定类型 `A` 的值 `a`，则很容易用 `flatMap` 实现：

```Scala
val aToB: A => F[B] = ???
val bToC: B => F[C] = ???

def example[A, C](a: A): F[C] =
  aToB(a).flatMap(bToC)
```

而 `Check` 中的 `andThen` 也可用于组合 `A => F[B]` 函数，因此上述转换也可用 `andThen` 实现：

```Scala
val aToC = aToB andThen bToC
```

* `aToC` 类型为 `A => F[C]` 函数，可以应用于类型 `A` 的值；

使用 `andThen` 可以在无 `A` 值时实现与 `flatMap` 类似的功能，这与函数组合的 `andThen` 函数非常类似，只不过两者组合的函数类型不同：

* `Check.andThen`：组合 `A => F[B]`
* `Function.andThen`：组合 `A => B`

`A => F[B]` 也是一种常用的函数类型，被称为 **Kleisli**。

Cats 中的 `ats.data.Kleisli` 与 `Check` 非常类似，包含所有 `Check` 中的函数，另外还有其他的有用函数，前面讲到的 `ReaderT` 就是 `Kleisli` 的别名。

## `Kleisili` 示例

下面是一个使用 `Kleisli` 的例子，下面 `step1` `step2` `step3` 每一步都将一个 `Int` 转换为一个 `List[Int]`：

```Scala
import cats.data.Kleisli
import cats.instances.list._

val step1: Kleisli[List, Int, Int] =
  Kleisli(n ⇒ List(n - 1, n + 1))

val step2: Kleisli[List, Int, Int] =
  Kleisli(n ⇒ List(n, -n))

val step3: Kleisli[List, Int, Int] =
  Kleisli(n ⇒ List(n * 2, n / 2))
```

用 `andThen` 将 `step1` `step2` `step3` 组合成 `pipeline`：

```Scala
val pipeline: Kleisli[List, Int, Int] = step1 andThen step2 andThen step3
```

`andThen` 底层是用 `flatMap` 实现的：

```Scala
def andThen[C](f: B => F[C])(implicit F: FlatMap[F]): Kleisli[F, A, C] =
  Kleisli((a: A) => F.flatMap(run(a))(f))
```

例子中的 `F` 为 `List`，因此实际使用时要导入 `Monad[List]` 实例：

```Scala
import cats.instances.list._

// xs: List[Int] = List(2, 0, -2, 0, 6, 1, -6, -1)
val xs: List[Int] = pipeline(2)
```

`Kleisli` 与 `Check` 在 API 方面唯一大的区别是，`Kleisli` 将 `apply` 命名为 `run`。

## 用 `Kleisli` 替换 `Check`

用 `Kleisli` 替换 `Check` 重写前面的校验例子，首先需要对 `Predicate` 做一点点修改：

1. 因为 `Kleisli` 只能用于函数，所以必须能将 `Predicate` 转换为函数；
2. 因为 `Kleisli` 底层依赖 `Monad`，因此 `Predicate` 转换后的函数不能是 `A => Validate[E, A]`（`Validate` 不是 `Monad`），而应该是 `A => Either[E, A]`；

### 实现 `run` 函数

为 `Predicate` 添加 `run` 函数，其类型应该是 `A => Either[E, A]`：

```Scala
sealed trait Predicate[E, A] {

  def run(implicit s: Semigroup[E]): A ⇒ Either[E, A] = a ⇒ this(a).toEither

  ...
}
```

与 `Predicate.apply` 类似，`run` 也需要接受 `implicit Semigroup` 参数，因为 `run` 内部是通过 `apply` 实现的。

### 替换 `Check`（未完成）

使用 `Kleisli` 和 `Predicate` 实现用户名、密码校验。