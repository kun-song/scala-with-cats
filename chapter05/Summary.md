# 5.5 Summary

对嵌套 monad 使用 for 解析、模式匹配时，for 解析和模式匹配也需要嵌套，使用不便，使用 monad transfomer 可以消除嵌套的 for 解析、模式匹配。

monad transformer 本身是一种数据结构，或者是一种 **容器**，用来包裹 monad stack，transformer 提供了 `map` `flatMap`，用于对 **整个 monad stack** 进行 unpack/repack。

例如 transformer `EitherT[Option, String, A]`，作为容器，它包裹的是 `Option[Either[String, A]]`。
