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

但 `Future` 并非引用透明的，所以不是展示 pure fp 的最佳例子。

## Functions 

实际上，只有 **一个** 参数的函数 `Function1` 也是 functor（惊不惊喜？），在 `Function1` 上执行 `map` 实质就是 **函数组合**：


```Scala
import cats.instances.function._
import cats.syntax.functor._

val f: Int ⇒ Int = x => x * 6
val g: Int => String = x => x + "!"

// 通过 map 实现的函数组合
(f map g)(1)

// 通过 andThen 实现的函数组合
(f andThen g)(1)

// 手撸的函数组合
(g(f(1))
```

`Function1` 本身代表了一次计算，每次调用 `map` 都会在原来基础上 **添加** 一层计算，最后得到的是 `sequence of computations`，到只调用 `map` 是不会有任何实际运算发生的。

最后给它一个实际参数，这才能使整个计算序列真正执行起来，这可以视为某种形式的惰性计算，类似 `Future`。
