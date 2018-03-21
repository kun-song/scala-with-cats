# Chapter 6 Semigroupal and Applicative

在前面 3 章中，介绍了用来 **序列化计算的** `map` 和 `flatMap` 函数，它们非常有用，但并非万能。

表单验证就无法用 `map` `flatMap` 实现，我们希望所有项校验完后，将所有错误结果发送给用户，若用 `map` `flatMap` 实现（例如 `Either`），则验证遇到 **第一个失败** 项就会停止，例如：

```Scala
import cats.syntax.either._

def parseInt(str: String): Either[String, Int] =
  Either.catchOnly[NumberFormatException](str.toInt)
    .leftMap(_ => s"Could not parse $str")

// scala.util.Either[String,Int] = Left(Could not parse a)
for {
  a ← parseInt("a")
  b ← parseInt("b")
  c ← parseInt("c")
} yield a + b + c
```
* 我们希望把 `a` `b` `c` 都校验完成后返回，实际遇到第一个无法解析的 `a` 就停止了；

另一个例子是 `Future` **并行计算**，若几个 `Future` 耗时较长，且 **无依赖关系**，则最好能并发执行它们。`map` 和 `flatMap` 无法实现该语义，因为 `map` `flatMap` 的基本 **前提假设**为：当前计算必然 **依赖** 前一个计算：

```Scala
// context2 依赖于 value1
context1.flatMap(value1 => context2)
```
* `map` 和 `flatMap` 都假定计算存在 **依赖** 关系，所以单纯用它们，无法实现并行；

这两个例子中对 `parseInt` 和 `Future.apply` 的调用是 **互相独立** 的，这是 `map` `flatMap` 无法表达的，因此需要一种 **不保证顺序** 的更弱的构建块，`Semigroupal` 和 `Applicative` 即为其中两种：

* `Semigroupal`
  + `Semigroupal` encompasses the notion of **composing pairs of contexts**.
  + Cats provides a `cats.syntax.apply` module that makes use of `Semigroupal` and `Functor` to allow users to sequence functions with **multiple arguments**.
* `Applicative` 
  + `Applicative` extends `Semigroupal` and `Functor`.
  + It provides a way of applying functions to parameters within a context. 
  + `Applicative` is the source of the `pure` method we introduced in Chapter 4.
