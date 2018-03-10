# 4.1 What is a Monad?

到底什么是 monad？

这确实是个大难题，有无数博客试图解答这个问题，但本书试图用非常简单的方式定义 monad：

>A monad is mechanism for sequencing computations.

这定义非常简单，但第三章讲 functor 的时候，貌似也是这样定义的！

第三章说 functor 允许我们在 **忽略某些复杂性** 的条件下 *sequence computations*，但 functor 有其局限性，即 functor 只允许 **某种复杂性** 在 *sequence of computations* 的 **开头** 出现一次！如果计算序列中的 **每次** 计算都会产生 **某种复杂性**，则 functor 就对此无能为力了。

monad 解决了 functor 的这种局限，monad 的 `flatMap` 函数会处理序列中 **每次计算** 涉及的 **某种复杂性**：

* `Option` 的 `flatMap` 将中间计算产生的 `Option` 考虑在内
* `List` 的 `flatMap` 将中间计算产生的 `List` 考虑在内

使用 `flatMap` 时，只需要把业务相关的函数传递给 `flatMap`，它会自动处理中间产生的 monad 实例，**每个 monad 实例都为用户隐藏了某种复杂性**，`flatMap` 可以让我们永远摆脱这些复杂细节：

### `Option`

`Option` allows us to sequence computations that **may or may not** return values，这里计算 **是否返回值** 就是 `Option` 试图为用户隐藏的 **复杂性**。

假设有如下两个函数：

```Scala
import scala.util.Try

def parseInt(v: String): Option[Int] =
  Try(v.toInt).toOption

def divide(x: Int, y: Int): Option[Int] =
  if (y == 0) None
  else Some(x / y)
```

`parseInt` 和 `divide` 返回值都是 `Option`，即他们都可能因为某种原因无法返回有意义的值，若想组合这两个函数，实现先从 `String` 解析出 `Int`，然后执行 `divide`（这是 **sequence computation** 的典型例子），应该怎么做呢，如果没有 `flatMap`，则只能通过手动 `if-else` 来处理每一步可能产生的 `None`，实现只有所有步骤都返回 `Some` 时才将计算继续下去：

```Scala
def stringDivide3(a: String, b: String): Option[Int] = {
  val x = parseInt(a)
  val y = parseInt(b)

  if (x.isEmpty || y.isEmpty) None
  else divide(x.get, y.get)
}
```

`stringDivide3` 中仅仅组合了两步计算，所以代码还能看懂，可涉及到更多计算步骤时，若还用手动 `if-else` 的方式处理，则要人命了。

这里每步计算都既可能有返回，也可能没有返回值，这种 **复杂性** 就是 `Option` 为我们隐藏的，使用 `flatMap` 可以避免直接处理这种复杂性：

```Scala
def stringDivide2(a: String, b: String): Option[Int] =
  parseInt(a).flatMap {
    x ⇒ parseInt(b).flatMap {
      y ⇒ divide(x, y)
    }
  }
```

`stringDivide2` 与 `stringDivide3` 语义完全相同：

* 若 `parseInt(a)` 返回 `None`，则 **立即** 结束计算，并返回 `None`，否则继续：
* 若 `parseInt(b)` 返回 `None`，则 **立即** 结束计算，并返回 `None`，否则继续：
* 执行 `divide`，返回其结果

```Scala
stringDivide2("20", "2")  // Some(10)
stringDivide2("20", "0")  // None
stringDivide2("xx", "2")  // None
stringDivide2("20", "x")  // None
```

>通过使用 `flatMap`，避免手动处理每次计算中可能出现的 `None`。

因为 **所有 monad 也都是 functor**，因此 monad 实例同时具备 `map` 和 `flatMap` 两个函数，使用原则如下：

* 若计算会产生 monad content，则用 `map`
* 若计算会产生 monad，则用 `flatMap`

并且，若 **同时具备** `map` 和 `flatMap`，则可以用 for 解析简化序列化计算：

```Scala
def stringDivide(a: String, b: String): Option[Int] =
  for {
    x ← parseInt(a)
    y ← parseInt(b)
    z ← divide(x, y)
  } yield z
```

`stringDivide` 非常清晰，Scala 编译器会将其转换为 `map` 和 `flatMap` 的实现，使用 for 解析可以很明确的表示 `parseInt(a)` `parseInt(b)` 和 `divide(x, y)` 之间的 **先后关系**。

### `List`

初学者看到 `List.flatMap` 时，往往会认为 `flatMap` 是一种列表遍历方式，for 解析尤其会强化这种想法：

```Scala
for {
  x ← (1 until 3).toList
  y ← (4 to 5).toList
} yield (x, y)
```

列表遍历还是太狭隘了，`List.flatMap` 行为本质依然是 monadic 的，上面的 for 解析是如下 3 个计算的序列化表示：

1. 获取 `x`
2. 获取 `y`
3. 创建 `(x, y)`

### `Future`

`Future` is a monad that sequences computations without worrying that they are asynchronous，**异步计算** 就是 `Future` 为用户隐藏的 **复杂性**：

```Scala
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._

def doSomethingLongRunning: Future[Int] = ???
def doSomethingElseLongRunning: Future[Int] = ???

def doSomethingVeryLongRunning: Future[Int] =
  for {
    result1 <- doSomethingLongRunning
    result2 <- doSomethingElseLongRunning
  } yield result1 + result2
```

使用 `Future.flatMap`，可以不再关注线程池、调度器等 **底层细节**，只需要表达我们要做什么就行了，上面表达的就是对异步计算的 `get` 语义。

注意，`doSomethingLongRunning` 和 `doSomethingElseLongRunning` 是顺序执行的，将其改写为 `map` + `flatMap` 后，顺序的本质更加清晰：

```Scala
def doSomethingVeryLongRunning: Future[Int] =
  doSomethingLongRunning.flatMap { result1 =>
    doSomethingElseLongRunning.map { result2 =>
      result1 + result2
    }
  }
```

计算序列中的每个 `Future`，都是通过一个函数创建的，例如：

```Scala
result1 =>
  doSomethingElseLongRunning.map { result2 =>
    result1 + result2
  }
```

该函数需要 `doSomethingLongRunning` 的 **返回值** 作为参数，因此必须等待 `doSomethingLongRunning` 完成后才能继续向下执行，以此类推，代表最后计算结果的 `Future` 是通过这个函数创建的：

```Scala
result2 => result1 + result2
```

以此最后的结果需要 **依次** 等待 `doSomethingLongRunning` 和 `doSomethingElseLongRunning` 计算完成。

## 4.1.1 Definition of a Monad

上面一直在将 `flatMap`，事实上，monadic behaviour 由两个操作组成：

* `pure` 函数，类型为 `A => F[A]`
  + `pure` 是对 constructor 的进一步抽象，可以通过 plain value 创建一个 monadic context（即 `F[_]`，即 `Option` `Future` 等）
* `flatMap` 函数，类型为 `(F[A], A => F[B]) => F[B]`
  + `flaMap` 从 monadic context 中提取值，然后产生一个新的 monadic context
  
下面是 Cats 中 `Monad` 的简化版：

```Scala
trait Monad[F[_]] {
  def pure[A](value: A): F[A]
  def flatMap[A, B](fa: F[A])(f: A ⇒ F[B]): F[B]
}
```

### Monad 法则

具备 `pure` 和 `flatMap` 仅仅是神似 monad，要变成真正的 monad，这两个函数必须满足 Monad 法则。

#### Left Identity

Calling `pure` and transforming the result with `f` is the same as calling `f`:

```Scala
pure(x).flatMap(f) == f(x)
```

#### Right Identity

Passing `pure` to `flatMap` is the same as doing nothing:

```Scala
m.flatMap(pure) == m
```

#### Associatity

`flatMapping` over two functions `f` and `g` is the same as `flatMapping` over `f` and then `flatMapping` over `g`:

```Scala
m.flatMap(f).flatMap(g) == m.flatMap(x => f(x).flatMap(g))
```

**注意**

* left 和 right 是 `pure` 相对 `flatMap` 的位置；

## 4.1.2 Exercise: Getting Func-y

所有 monad 都是 functor，可以用 `flatMap` 和 `pure` 实现 `map`，根据类型进行推导，只有一种实现方式：

```Scala
trait Monad[F[_]] {
  def pure[A](value: A): F[A]
  def flatMap[A, B](fa: F[A])(f: A ⇒ F[B]): F[B]

  def map[A, B](fa: F[A])(f: A ⇒ B): F[B] =
    flatMap(fa)(a ⇒ pure(f(a)))
}
```
