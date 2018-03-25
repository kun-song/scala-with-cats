# Chapter 7 Foldable and Traverse

本章介绍个两个用于 **iteration over collections** 的 type class：

1. `Foldable`
  * 抽象于 `foldLeft` 和 `foldRight`
2. `Traverse`
  * 抽象层次比 `Foldable` 更高
  * 有的场景用 `Foldable` 遍历很繁琐，`Traverse` 用于简化这些场景的遍历

首先介绍 `Foldable`，然后看几个使用 `Foldable` 非常繁琐，而用 `Traverse` 非常简洁的例子。