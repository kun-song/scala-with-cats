# 5.1 Exercise: Composing Monads

问题来了，给定两个 monad 实例，是否能以某种方式，将它们 **组合** 为一个 monad 呢？

```Scala
import cats.Monad
import cats.syntax.applicative._

import scala.language.higherKinds

def compose[M1[_]: Monad, M2[_]: Monad] = {

  type Composed[A] = M1[M2[A]]

  new Monad[Composed] {
    override def pure[A](x: A) = x.pure[M2].pure[M1]

    override def flatMap[A, B](fa: Composed[A])(f: A ⇒ Composed[B]) = ???

    override def tailRecM[A, B](a: A)(f: A ⇒ Composed[Either[A, B]]) = ???
  }
}
```

`pure` 实现很简单，但是 `flatMap` 呢？

事实上，`M1` 和 `M2` 都是非常 general 的 monadic context，若对 `M1` `M2` 一无所知，则根本不可能为 `M1[M2]` 实现 `flatMap`。

而若 `M1` `M2` 任意一个 **确定下来**，例如若 `M2` 为 `Option`，则 `flatMap` 就很容易实现了：

```Scala
def flatMap[A, B](fa: Composed[A])(f: A => Composed[B]): Composed[B] =
  fa.flatMap(_.fold(None.pure[M])(f))
``` 

上面的 `flatMap` 实现利用了 `None`，而 `None` 是 `Option` 中的概念，而非 `Monad` 中的通用概念。将 `Option` 与其他 `Monad` 组合时，我们需要借助 `None` 这点 **额外信息** 来帮助我们实现组合。

类似，其他 `Monad` 也 **各自** 有自己的 **额外信息**，可用来实现自己与其他 `Monad` 的 **组合**，这就是 monad transformer 背后的思想，Cats 为很多 `Monad` 定义了 monad transformer，每个 transformer 都为特定 `Monad` 提供了 **extra knowledge**，用来实现自己与其他 monad 的组合。