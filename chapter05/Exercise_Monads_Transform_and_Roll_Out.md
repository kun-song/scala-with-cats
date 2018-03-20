# 5.4 Exercise: Monads: Transform and Roll Out

汽车人在战斗中需要互相频繁发送消息，获取战友的能量水平，以协调战术、发动攻击，消息发送函数签名如下：

```Scala
def getPowerLevel(autobot: String): Response[Int] = ???
```

消息可能由于各种各样的原因发送失败，因此 `Response` 是一个 monad stack：

```Scala
type Response[A] = Future[Either[String, A]]
```

要获取 `Response[A]` 中的消息 `A`，需要用嵌套的 for 解析（或嵌套的 `flatMap` + `map`），擎天柱厌烦了各种嵌套，请用 monad transformer 重写 `Response`，以消除 for 嵌套：

```Scala
import cats.data.EitherT
import scala.concurrent.Future

type Response[A] = EitherT[Future, String, A]
```

假设能力数据如下：

```Scala
val powerLevels = Map(
  "Jazz"      -> 6,
  "Bumblebee" -> 8,
  "Hot Rod"   -> 10
)
```

实现 `getPowerLevel` 函数，注意若查询的名字不在数据库中，则返回的消息中需要说明其 unreachable，并附带其名字：

```Scala
import cats.data.EitherT
import cats.instances.future._
import cats.syntax.applicative._

import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

def getPowerLevel(autobot: String): Response[Int] =
  powerLevels.get(autobot) match {
    case Some(x)  ⇒ x.pure[Response]
    case None     ⇒ EitherT.left(Future(s"$autobot unreachable"))
  }
```

若给定的名字无法查到，会体现在返回值中：

```Scala
// Response[Int] = EitherT(Future(Success(Left(xxx unreachable))))
getPowerLevel("xxx")
```

若两个汽车人的能量和超过 15，就可以实现一个特殊移动，实现 `def canSpecialMove(ally1: String, ally2: String): Response[Boolean]` 函数，接受两个名字，检查其是否可特殊移动，若任意一方 unreachable，需要在返回消息中体现：

```Scala
def canSpecialMove(ally1: String, ally2: String): Response[Boolean] =
  for {
    a ← getPowerLevel(ally1)
    b ← getPowerLevel(ally2)
  } yield (a + b) > 15
```

若任意给定名字无法查到：

```Scala
// Either[String,Boolean] = Left(xxx unreachable)
Await.result(canSpecialMove("Jazz", "xxx").value, 1.second)
```

最后实现 `tacticalReport` 函数，接受两个名字，返回字符串，说明它们是否可以特殊移动：

```Scala
def tacticalReport(ally1: String, ally2: String): String =
  Await.result(canSpecialMove(ally1, ally2).value, 1.second) match {
    case Right(true)  ⇒ s"$ally1 and $ally2 are ready to roll out!"
    case Right(false) ⇒ s"$ally1 and $ally2 need a recharge."
    case Left(msg)    ⇒ s"Comms error: $msg"
  }
```

使用如下：

```Scala
tacticalReport("Jazz", "Bumblebee")
// res28: String = Jazz and Bumblebee need a recharge.

tacticalReport("Bumblebee", "Hot Rod")
// res29: String = Bumblebee and Hot Rod are ready to roll out!

tacticalReport("Jazz", "Ironhide")
// res30: String = Comms error: Ironhide unreachable
```