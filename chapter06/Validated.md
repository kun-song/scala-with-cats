# 6.4 Validated

前面展示了以 `Either` 实现 fail-fast error-handling，因为 `Either` 是 `Monad`，因此其 `product` 与 `flatMap` 语义完全相同。

更进一步，由于 Monad 的 `product` 是用 `flatMap` 实现的，因此使用 `Monad` 根本无法实现 accumulating error-handling！

>注意，因为 `Monad` 继承了 `Semigroupal`，因此对于类型 `A`，只要实现 `Monad[A]` 实例即可，不需要单独实现 `Semigroupal[A]` 实例。

幸运的是 Cats 提供了 `Validated` 类型，且对该类型，只有 `Semigroupal[Validated]` 实例，而无 `Monad[Validated]` 实例，因此 `Semigroupal[Validated]` 可以自由地实现 accumulating 的 `product` 函数：

```Scala

```