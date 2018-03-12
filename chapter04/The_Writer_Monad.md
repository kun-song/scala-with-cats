# 4.7 The Writer Monad

`cats.data.Writer` monad 可以携带 log 和 computation，可以用于记录相关计算的消息、日志、错误等，等所有计算结束，可以从 `Writer` 中提取日志和计算结果。

`Writer` 的一个典型应用场景是：多线程计算中，传统的日志记录方式可能导致 **重叠** 的日志，使用 `Writer` 可以实现与 **计算顺序** 一致的顺序日志。

## 4.7.1 Creating and Unpacking Writers

`Writer[W, A]` 携带两个值：类型为 `W` 的日志 + 类型为 `A` 的计算，可以从 `W` 和 `A` 的值创建 `Writer`：

```Scala
import cats.data.Writer
import cats.instances.vector._

// WriterT[cats.Id,Vector[String],Int] = WriterT((Vector(step 1, step 2),666))
Writer(
  Vector("step 1", "step 2"),
  666
)
```

注意返回值类型为 `WriterT[Id, Vector[String], Int]` 而非 `Writer[Vector[String], Int]`，原因是，为了代码复用，Cats 的 `Writer` 其实是通过 `WriterT` 实现的，而 `WriterT` 是 monad transformer 的一个例子。

`Writer` 实际是 `WriterT` 的别名：

```Scala
type Writer[W, A] = WriterT[Id, W, A]
```

为方便起见，可以分别从 log 或者 computation 创建 `Writer`。

若只有 result，则可用 `pure` syntax（`cats.syntax.applicative._`） 创建 `Writer`，但要求 `implicit` 作用域中有 `Writer[W]` 实例，用于创建空的 log：

```Scala
import cats.data.Writer
import cats.instances.vector._
import cats.syntax.applicative._

type Log[A] = Writer[Vector[String], A]

111.pure[Log]
```
* 通过 `Log[A] = Writer[Vector[String], A]` 简化 `pure` 的写法，否则需要写成 `111.pure[Writer[Vector[String], Int]]`
* 本例中 `W` 为 `Vector[String]`，所以需要 `Monad[Vector[String]]` 实例，通过 `cats.instances.vector._` 导入

若只有 log 而无 result，则可通过 `tell` syntax（`cats.syntax.writer._`）创建 `Writer[W, Unit]`：

```Scala
import cats.syntax.writer._

// Writer[Vector[String],Unit]
Vector("hello", "world").tell
```

若同时有 log 和 result，则即可用 `Writer.apply`，也可用 `writer` syntax 创建：

```Scala
import cats.data.Writer
import cats.syntax.writer._

Writer(
  Vector("step 1", "step 2"),
  666
)

val a = 666 writer Vector("m1", "m2")
```

可分别通过 `value` 和 `written` 提取 `Writer` 中的 result 和 log：

```Scala
import cats.syntax.writer._

val a = 666 writer Vector("m1", "m2")

val value = a.value
val log = a.written
```

也可用 `run` 同时提取 result 和 log：

```Scala
val (log, result) = a.run
```

## 4.7.2 Composing and Transforming Writers

可用 `map` 和 `flatMap` 对 result 进行转换，但 `Writer` 中的 log 将被保持，例如 `flatMap` 会将上个 `Writer` 的 log 与本次产生的 `Writer` 的 log 做 **append** 操作，因此最好为 log 选一个 append 效率高的数据类型，例如 `Vector`：

```Scala
import cats.data.Writer
import cats.instances.vector._
import cats.syntax.applicative._
import cats.syntax.writer._


type Log[A] = Writer[Vector[String], A]

/**
  * WriterT((Vector(a, b, c, x, y, z),666))
  */
val w1 =
  for {
    a ← 111.pure[Log]
    _ ← Vector("a", "b", "c").tell
    b ← 6 writer Vector("x", "y", "z")
  } yield a * b
```

可用 `mapWritten` 对 log 进行转换，


```
// WriterT((Vector(A, B, C, X, Y, Z),666))
w1.mapWritten(_.map(_.toUpperCase))
```

可用 `mapBoth` 和 `bimap` 同时对 log 和 result 转换：

```Scala
/**
  * WriterT((Vector(a!, b!, c!, x!, y!, z!),6660))
  */
w1.mapBoth{
  (log, result) ⇒ {
    val l = log.map(_ + "!")
    val r = result * 10
    (l, r)
  }
}

/**
  * WriterT((Vector(a!, b!, c!, x!, y!, z!),6660))
  */
w1.bimap(
  log ⇒ log.map(_ + "!"),
  result ⇒ result * 10
)
```
* `mapBoth` 接受一个函数参数
* `bimap` 接受两个函数参数

可用 `reset` 清除 log：

```Scala
/**
  * WriterT((Vector(),666))
  */
w1.reset
```

可用 `swap` 交换 log 和 result：

```Scala
/**
  * WriterT((666,Vector(a, b, c, x, y, z)))
  */
w1.swap
```

## 4.7.3 Exercise: Show Your Working

前面说过 `Writer` 可以用于多线程场景中 *顺序** 打印日志，现在如下求斐波那契数列的函数：

```Scala
def slowly[A](body: => A) =
  try body finally Thread.sleep(100)

def factorial(n: Int): Int = {
  val ans = slowly(if(n == 0) 1 else n * factorial(n - 1))
  println(s"fact $n $ans")
  ans
}
```

在多线程中，无法分辨 `println` 输出到底是属于哪个线程：

```Scala
/**
  * fact 0 1
  * fact 0 1
  * fact 1 1
  * fact 1 1
  * fact 2 2
  * fact 2 2
  * fact 3 6
  */
Await.result(Future.sequence(Vector(
  Future(factorial(2)),
  Future(factorial(3))
)), 5.seconds)
```

使用 `Writer` 改写如下：

```Scala
def factorial(n: Int): Writer[Vector[String], Int] = {
  slowly(
    if(n == 0) 1 writer Vector("fact 0 1")
    else {
      factorial(n - 1).mapBoth((w, result) ⇒ {
        val ans = result * n
        val log = w :+  s"fact $n $ans")
        (log, ans)
      })
    }
  )
}
```

此时使用 `Vector[String]` 保存日志：

```Scala
/**
  * Vector(WriterT((Vector(fact 0 1, fact 1 1, fact 2 2),2)), 
  *        WriterT((Vector(fact 0 1, fact 1 1, fact 2 2, fact 3 6),6)))
  */
Await.result(Future.sequence(Vector(
  Future(factorial(2)),
  Future(factorial(3))
)), 5.seconds)
```

现在虽然满足题目要求，但是实现却有点繁琐，用 for 解析改写：

```Scala
import cats.data.Writer
import cats.syntax.writer._
import cats.syntax.applicative._

type Log[A] = Writer[Vector[String], A]

def factorial(n: Int): Writer[Vector[String], Int] =
  for {
    ans ← if (n == 0) 1.pure[Log] else slowly(factorial(n - 1).map(_ * n))
    _   ← Vector(s"fact $n $ans").tell
  } yield ans
```
