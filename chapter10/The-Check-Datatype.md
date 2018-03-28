# 10.2 The Check Datatype

我们的设计是围绕 check 展开的，前面说过 check 应该是一个 `A => F[A]` 的函数，最容易想到的定义是：

```Scala
type Check[A] = A => Either[String, A]
```
* `Check` 类型为 `A => Either[String, A]`

这里使用 `String` 表示错误提示信息，但用 `List` 似乎更好（`List` 能收集所有错误信息），甚至可用支持国际化的某种类型，更甚至可以用错误码！

可以定义一个 `ErrorMessage`，来兼容 **所有** 可能出现的类型，但为何不将这交给用户决定呢？因此可以为 `Check` 增加一个类型参数：

```Scala
type Check[E, A] = A => Either[E, A]
```

以后极有可能需要为 `Check` 添加函数，因此将其定义为特质：

```Scala
trait Check[E, A] {
  def apply(value: A): Either[E, A]
}
```

正如 Essential Scala 中讲到，使用 `trait` 时主要有两种模式：

1. 将 `trait` 定义为 type class
2. 将 `trait` 定义为 algebraic data type（此时一般用 `sealed trait`）

使用 type class 主要是为 **差异巨大** 的数据类型定义 **统一** 的接口，这并不适合此时的场景，因此需要将 check 定义为 algebraic data type，后面设计过程中，谨记此事。
