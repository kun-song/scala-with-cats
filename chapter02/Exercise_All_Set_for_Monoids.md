# 2.4 Exercise: All Set for Monoids

对于 `Set` 类型，可以定义哪些 `Monoid` 和 `Semigroup` 实例呢？

首先，`Set` 有 4 种典型操作：

* `union` 取并集
* `intersect` 取交集
* `diff` 取差 + `complement` 取补集

### 1. `union`

取并集可以定义为 `Monoid`，单位元为空集合：

```Scala
implicit def unionMonoid[A]: Monoid[Set[A]] =
  new Monoid[Set[A]] {
    override def empty: Set[A] = Set.empty[A]
    override def combine(x: Set[A], y: Set[A]): Set[A] = x union y
  }
```

证明：

```Scala
associativeLaw(Set(1, 2, 3), Set(4, 5), Set(6))(unionMonoid)

identityLaw(Set(1, 2, 3))(unionMonoid)
```

这里使用 `def` 定义 `unionMonoid`，而非 `val`，原因是 `def` 定义可以接受类型参数，使用 `val` 会将 `unoinMonoid` 限制在某一特定类型的 `Set` 上。

可以这样使用 `unionMonoid`：

```Scala
val intSetMonoid: Monoid[Set[Int]] = Monoid[Set[Int]]

intSetMonoid.combine(Set(1, 2, 3), Set(1, 2, 3, 4, 5))
```

### 2. `intersect`

取交集只能定义为 `Semigroup`，因为无法为它找到有意义的 *单位元*：

```Scala
implicit def insersectSemigroup[A]: Semigroup[Set[A]] =
  (x, y) => x intersect y
```

### 3. `complement` 和 `difference`

差集和补集都不满足结合律，因此不能为其定义 `Monoid` 或 `Semigroup` 实例。