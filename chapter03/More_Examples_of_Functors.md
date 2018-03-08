# 3.2 More Examples of Functors

前面讲的具备 `map` 的 `List` `Option` 等都比较简单，下面介绍可能让人别开生面的 `map` 应用方式。

## Futures

`Future` 用于 **序列化异步计算**，这是通过将计算放到 *队列* 中，然后依次应用实现的。

使用 `Future` 时，其内部包裹的计算可能处于进行中、完成或被拒绝等状态，但只有当 `Future` **完成** 时才会调用 `map`，否则底层的线程池会将 `map` 调用排队，等 `Future` 完成后再调用。

因此无法确定 `Future.map` *何时* 执行，但是多个 `map` 调用的执行 *顺序* 是确定的，因此 `map` 的行为与 `List` 上并无区别：

```Scala
import scala.concurrent.{Await, Future}
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._

val f: Future[String] =
  Future(110)
    .map(_ + 1)
    .map(_ * 6)
    .map(_ + "!")

Await.result(f, 1.second)  // 666!
```
