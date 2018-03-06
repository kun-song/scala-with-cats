# 2.2 Definition of a Semigroup

semigroup 是 monoid 中的 `combine` 部分，剔除掉了 `empty` 单位元。

虽然有很多 semigroup 也是 monoid，但有些数据类型，没办法定义有意义的 `empty` 单位元。

例如，虽然序列拼接、整数相加都是 monoid，但假设增加一点限制：非空序列拼接、正整数相加，现在就无法定义 `empty` 单位元了，因为它根本不存在！

所以我们需要 semigroup，它更加通用，适用于刚才的场景。

`Semigroup` 可以定义为：

```Scala
trait Semigroup[A] {
  def combine(x: A, y: A): A
}
```

而 `Monoid` 可视为 `Semigroup` 的子类：

```Scala
trait Monoid[A] extends Semigroup[A] {
  def empty: A
}
```

因此，若为类型 `A` 定义了 `Monoid[A]`，自然也就得到了一个 `Semigroup[A]`。