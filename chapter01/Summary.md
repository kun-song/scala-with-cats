# 1.7 Summary

本章首先实现了一个 type class `Printable`，然后介绍了两个 Cats 中的 type class，即 `Eq` 和 `Show`。

至此，Cats type class 的基本套路已经很清晰了：

* type class 本身被定义为 `trait`
* 每个 type class 的伴生对象中都有：
  1. `apply` 方法，用于定位指定类型的 instance
  2. 1 个或多个辅助构造方法，用于创建 instance
  3. 其他辅助方法
* 默认 instances 在 `cats.instances` 包中
* 很多 type class 具备 interface syntanx，在 `cats.syntanx` 包中

下面将介绍一些更加强大的 type class，包括 `Semigroup` `Monoid` `Functor` `Monad` `Semigroupal` `Applicative` 和 `Traverse` 等。在介绍每个 type class 时，包含以下内容：

* 该 type class 提供的 **功能**
* 该 type class 遵守的 **法则**
* 该 type class 在 Cats 中是如何 **实现** 的

剩余的 type class 要比 `Eq` `Show` 更加抽象，从而更加通用，也更加难以理解。
