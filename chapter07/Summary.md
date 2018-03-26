# 7.3 Summary

本章介绍两个用于序列元素遍历的 type class：

* `Foldable`
  + 从 `foldLeft` 和 `foldRight` 抽象而来
  + `Foldable.foldRight` 是栈安全的
  + `Foldable` 定义了一些有用函数
* `Traverse`
  + `F[G[A]]` -> `G[F[A]]`
    - `F` 有 `Traverse` 实例
    - `G` 有 `Applicative` 实例
  + 将多行 `fold` 代码简化为一行 `traverse`
  
