# Chapter 4 Monads

Monad 可能是 Scala 中最常见的抽象了。

不正式的讲，**a monad is anything with a constructor and a `flatMap` method**，第三章介绍的那些 functor，例如 `Option` `List` 和 `Future` 等，既是 functor，也是 monad。

虽然 monad 概念如此广泛，但 Scala 语言并未内置，幸好 Cats 把 Monad 带给了 Scala 教徒。