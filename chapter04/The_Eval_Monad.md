# 4.6 The Eval Monad

`cats.Eval` 是对不同 **evaluation models** 抽象出的 monad。

我们一般都听说过 eager 和 lazy 两种求值模型，而 `Eval` 更进一步，对结果是否缓存做了区分（即 memoized）。

## 4.6.1 Eager, Lazy, Momoized

3 者区别如下：

* eager computations
  + 立即求值
* lazy computations
  + access 时求值
* memoized computations
  + 在 first access 时求值一次，之后缓存求值结果

例如，Scala 中的 `val` 是 eager + memoized：

```Scala
val x = {
  println("compute x")
   666
}

println(s"1: $x")
println(s"2: $x")
println(s"3: $x")
```

输出：

```Scala
compute x
1: 666
2: 666
3: 666
```
* 仅定义时求值一次

Scala 中的 `def` 则是 lazy，且 non-memoized：

```Scala
def x = {
  println("compute x")
  666
}

println(s"1: $x")
println(s"2: $x")
println(s"3: $x")
```

输出：

```Scala
compute x
1: 666
compute x
2: 666
compute x
3: 666
```
* 每次引用，重复求值

Scala 中的 `lazy val` 则是 lazy + memoized：

```Scala
lazy val x = {
  println("compute x")
  666
}

println(s"1: $x")
println(s"2: $x")
println(s"3: $x")
```

输出：

```Scala
compute x
1: 666
2: 666
3: 666
```
* 首次访求，求值一次

## `Eval`'s Models of Evaluation

`Eval` 有 3 个子类：

* `Now`
* `Later`
* `Always`

分别有一个构造器用于构造这 3 个子类的实例，参数为 **表达式**，以 `Eval` 类型返回它们：

* `Eval.now()`
* `Eval.later()`
* `Eval.always()`

通过 `Eval.value` 函数提取 `Eval` 中的计算结果：

```Scala
import cats.Eval

import scala.util.Random

// 定义时求值
val now = Eval.now("now: " + Random.nextInt)
// 首次访问求值
val later = Eval.later("later: " + Random.nextInt)
// 每次访问都求值
val always = Eval.always("always: " + Random.nextInt)

now.value  // 不再求值
now.value  // 不再求值

later.value  // 首次访问求值
later.value  // 不再求值

always.value  // 求值
always.value  // 求值
```
* `now.value` 和 `later.value` 只求值一次，因此多次调用 `value` 结果相同
* `always.value` 每次调用都会求值，每次结果不同

Cats 中的 `Now` `Later` `Always` 分别与 Scala 中的 `val` `lazy val` `def` 一一对应：

| Scala | Cats | 属性 |
| :---: | :----: | :----: |
| `val` | `Now` | eager + memoized |
| `lazy val` | `Later` | lazy + memoized |
| `def` | `Always` | lazy |

## 4.6.3 `Eval` as a Monad

与其他 monad 相同，`Eval` 的 `map` 和 `flatMap` 都会向计算链中 **添加计算**，`Eval` 的计算链是以显式的函数列表保存的，计算链中的函数 **不会** 立即执行，只有调用 `Eval.value` 时才会 **真正执行计算**：

```Scala
import cats.Eval

val greetings = Eval.
  always {
    println("step 1")
    "hello"
  }.
  map { s ⇒
    println("step 2")
    s"$s world!"
  }

// step 1
// step 2
// hello world!
greetings.value
```

注意，使用 `Eval.now/later/always` 时，求值规则肯定与 `now` `later` `always` 的规则吻合，但是 `map` 中计算 **永远是 `lazy`** 的：

```Scala
import cats.Eval

// calculating a
val ans =
  for {
    a ← Eval.now { println("calculating a"); 111 }
    b ← Eval.always { println("calculating b"); 6 }
  } yield {
    println("add a and b")
    a * b
  }

// calculating b
// add a and b
// 666
ans.value  // 首次访问

// calculating b
// add a and b
// 666
ans.value  // 第二次访问
```

`Eval.memoize` 用于 **缓存** a chain of computations，从创建开始，直到 `memoize` 调用为止，中间的计算链的结果都会被缓存，而 `memoize` 之后的其他计算则 **不受影响**：

```Scala
import cats.Eval

val greetings = Eval.
  always {
    println("step 1")
    "hello"
  }.
  map { s ⇒
    println("step 2")
    s"$s world"
  }.
  memoize.
  map { s ⇒
    println("step 3")
    s"$s!!!"
  }

/**
  * step 1
  * step 2
  * step 3
  * res0: String = hello world!!!
  */
greetings.value

/**
  * step 3
  * res1: String = hello world!!!
  */
greetings.value
```

可以看到，第二次调用 `Eval.value` 时，`memoize` 之前的计算并未重复计算。

## 4.6.4 Trampolining and `Eval.defer`

`Eval.map` 和 `Eval.flatMap` 都是 **trampolined** 的，即可以任意嵌套使用 `map` 和 `flatMap`，而不会消耗栈帧，该属性被称为 *stack safety*。

Scala 是 fp 语言，崇尚以 **递归** 解决问题，例如：

```Scala
def factorial(n: BigInt): BigInt =
  if (n == 0) 1
  else n * factorial(n - 1)
```

上面求斐波那契数列的函数非常符合直觉，但在 `n` 很大时，非常容易出现栈溢出：

```Scala
factorial(10000)  // 抛出 java.lang.StackOverflowError
```

可以用 `Eval` 改写该函数，以实现 stack safety：

```Scala
def factorial(n: BigInt): Eval[BigInt] =
  if (n == 0) Eval.now(1)
  else factorial(n - 1).map(_ * n)
```

但此时计算 `factorial(10000)` 依然抛出 `StackOverflowError`，这是咋回事呢？

原因很简单，前面说了 `Eval.map` 和 `Eval.flatMap` 是 trampolined，但上面例子中，在执行 `Eval.map` 之前，先进行了 `factorial` 的递归调用，等于是还没等到 `map` 大展神威，`factorial` 的递归调用已经耗尽了栈空间。

解决方式是使用 `Eval.defer`，该函数接受一个 `Eval` 实例作为参数，并 **延迟** 该 `Eval` 实例的计算，且与 `map` `flatMap` 相同，`defer` 也是 trampolined，可以放心使用：

```Scala
import cats.Eval

def factorial(n: BigInt): Eval[BigInt] =
  if (n == 0) Eval.now(1)
  else Eval.defer(factorial(n - 1).map(_ * n))

factorial(10000000).value
```
* 使用 `Eval.defer` 包裹整个 `factorial(n - 1).map(_ * n)`（我觉得只包裹 `factorial(n - 1)` 也行）

>`Eval` 可以实现栈安全的计算，但 `Eval` 并非是万能的，它通过在 heap 上创建一系列的函数对象来 **避免** stack frame 的创建，但因为 heap 大小也是有限的，所以 `Eval` 支持的嵌套层次也是有限的，不过 heap 要比 stack 大很多，所以层次要比改良之前深很多。

## 4.6.5 Exercise: Safer Folding using `Eval`

以下是 `foldRight` 的一个简单实现，它并非栈安全的，使用 `Eval` 将其改写为栈安全版本：

```Scala
def foldRight[A, B](xs: List[A], acc: B)(f: (A, B) ⇒ B): B =
  xs match {
    case hd :: tl ⇒ f(hd, foldRight(tl, acc)(f))
    case Nil      ⇒ acc
  }

foldRight(List.fill(10000)(10), 0)((a, b) ⇒ a + b)  // 抛出 StackOverflowError
```

改写后：

```Scala
def foldRight[A, B](xs: List[A], acc: B)(f: (A, B) ⇒ B): Eval[B] =
  xs match {
    case hd :: tl ⇒ Eval.defer(foldRight(tl, acc)(f).map(b ⇒ f(hd, b)))
    case Nil      ⇒ Eval.now(acc)
  }

foldRight(List.fill(10000)(10), 0)((a, b) ⇒ a + b).value  // 不再栈溢出
```