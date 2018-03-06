# 2.3 Exercise: The Truth About Monoids

世界上的 monoid 例子举不胜举，例如对于 `Boolean` 类型，就可以定义多个 `Monoid[Boolean]` 实例.

```Scala
trait Semigroup[A] {
  def combine(x: A, y: A): A
}

trait Monoid[A] extends Semigroup[A] {
  def empty: A
}

object Monoid {
  def apply[A](implicit m: Monoid[A]): Monoid[A] = m
}
```

### 1. `and`

`Boolean` 的 `&&` 可以定义一个 monoid，其操作符即 `&&`，单位元为 `true`：

```Scala
implicit val andMonoid: Monoid[Boolean] =
  new Monoid[Boolean] {
    override def empty: Boolean = true
    override def combine(x: Boolean, y: Boolean): Boolean = x && y
  }
```

证明 `andMonoid` 满足结合律、单位元法则：

```Scala
associativeLaw(true, false, true)(andMonoid)  // true

identityLaw(true)(andMonoid)  // true
identityLaw(false)(andMonoid)  // true
```

### 2. `or`

`or` 也是一个 monoid，其操作符为 `||`，单位元为 `false`：

```Scala
implicit val orMonoid: Monoid[Boolean] =
  new Monoid[Boolean] {
    override def empty: Boolean = false
    override def combine(x: Boolean, y: Boolean): Boolean = x || y
  }
```

证明 `orMonoid` 满足结合律、单位元法则：

```Scala
associativeLaw(true, false, true)(orMonoid)  // true

identityLaw(true)(orMonoid)  // true
identityLaw(false)(orMonoid)  // true
```

### 3. exclusive `or`

单位元为 `false`：

```Scala
implicit val eitherMonoid: Monoid[Boolean] =
  new Monoid[Boolean] {
    override def empty: Boolean = false
    override def combine(x: Boolean, y: Boolean): Boolean =
      (x && !y) || (!x && y)
  }
```

证明 `eitherMonoid` 满足结合律、单位元法则：

```Scala
associativeLaw(true, false, true)(eitherMonoid)

identityLaw(true)(eitherMonoid)
identityLaw(false)(eitherMonoid)
```

### 3. exclusive `nor`(the negative of exclusive `or`)

单位元为 `true`：

```Scala
implicit val norMonoid: Monoid[Boolean] =
  new Monoid[Boolean] {
    override def empty: Boolean = true
    override def combine(x: Boolean, y: Boolean): Boolean =
      (!x || y) && (x || !y)
  }
```

证明 `norMonoid` 满足结合律、单位元法则：

```Scala
associativeLaw(true, false, true)(norMonoid)

identityLaw(true)(norMonoid)
identityLaw(false)(norMonoid)
```

>注意：
>
>1. 针对 `Boolean` 类型，竟然有 4 种 `Monoid` 实例！
>2. 上面对单位元法则的证明是完备的，但是对结合律的证明应该枚举 3 个参数所有的组合情况