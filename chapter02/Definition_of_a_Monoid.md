# 2.1 Definition of a Monoid

前面介绍了很多 **相加** 的例子，它们都包含：一个二元加操作 & 一个单位元，这就是一个 monoid，正经点讲，一个针对类型 `A` 的 monoid 是：

* 二元操作 `combine`，类型为 `(A, A) => A`
* 单位元 `empty`，类型为 `A`

转换为 Scala 代码为：

```Scala
trait Monoid[A] {
  def combine(x: A, y: A): A
  def empty: A
}
```

除了定义中的 `combine` 和 `empty` 方法外，`Monoid` 还需要满足 2 个法则：

### 结合律

`combine` 必须满足结合律：

```Scala
def associativeLaw[A](x: A, y: A, z: A)(implicit m: Monoid[A]): Boolean =
  m.combine(m.combine(x, y), z) == m.combine(x, m.combine(y, z))
```

### 单位元法则

`empty` 必须满足单位元法则：

```Scala
def identityLaw[A](x: A)(implicit m: Monoid[A]): Boolean =
  m.combine(x, m.empty) == x && m.combine(m.empty, x) == x
```

实践中，只有实现自己的 `Monoid` instances 时才需要考虑上面的法则，而大多数场景中，我们可以直接使用 Cats 提供的 instances，这时由 Cats 的设计者负责其提供的 instances 满足以上法则。
