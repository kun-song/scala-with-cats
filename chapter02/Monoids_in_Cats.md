# 2.5 Monoids in Cats

前面都是手撸的 `Monoid`，`Monoid` 在 Cats 中是怎么实现的呢？`Monoid` 是 type class，所以我们需要了解其三大组成部分（type class + instance + interface syntax）分别是如何实现的。 

## 2.5.1 The `Monoid` Type Class

Cats 的 `Monoid` type class 为 `cats.kernel.Monoid`，为方便使用，`cats.Monoid` 是其同义词，`Monoid` 继承了 `cats.kernel.Semigroup`（同义词为 `cats.Semigroup`）：

```Scala
import cats.Monoid
import cats.Semigroup
```

>Cats Kernel 是什么？
>
>Cats Kernel 是 Cats 的子项目，提供一个 type class 的最小集合，虽然这些 type class 定义在 `cats.kernel` 包中，它们全部都有 `cats.xx` 这样的同义词，因此，从使用者角度看，并不需要关系 Cats Kernel 的存在。
>
>本书涉及的 Cats Kernel 中的 type class 为 `Eq` `Semigroup` 和 `Monoid`，其余 type class 均直接定义在 `cats.` 中。

## 2.5.2 Monoid Instances

前面说过，在 Cats 中，type class 的伴生对象中有一个 `apply` 方法，该方法会返回指定类型的 type class instance。例如 `Monoid[String]` 将返回一个 `Monoid[String]` 实例：

```Scala
import cats.Monoid
import cats.instances.string._

Monoid[String].combine("hello", "world")

Monoid[String].empty
```

注意 `Monoid[String].empty` 中的 `Monoid[String]` 是一个方法调用，它没有任何参数，是 `Monoid[String].apply` 方法的简写，因此下面的例子与上面的完全等价：

```Scala
import cats.Monoid
import cats.instances.string._

Monoid.apply[String].combine("hello", "world")

Monoid.apply[String].empty
```

因为 `Monoid` 继承了 `Semigroup`，所以如果不需要单位元，则可以导入 `Semigroup` type class：

```Scala
import cats.Semigroup
import cats.instances.string._

Semigroup.apply[String].combine("hello", "world")
```

同样，可以通过 `Monoid[Option]` 和 `Monoid[Int]` 组合出 `Monoid[Option[Int]]`：

```Scala
import cats.Monoid
import cats.instances.int._
import cats.instances.option._

val a = Option(1)
val b = Option(2)

Monoid[Option[Int]].combine(a, b)
```

## 2.5.3 Monoid Syntax

Cats 为 `combine` 函数定义了友好的操作符 `|+|`，`|+|` 定义在 `cats.syntax.semigroup` 中，但因为 `cats.syntax.monoid` 继承了前者，所以导入二者任一即可：

```Scala
import cats.instances.int._
import cats.instances.option._
import cats.syntax.semigroup._
// or cats.syntax.monoid._

val a = Option(1)
val b = Option(2)

a |+| b
a combine b

1 combine 2
1 |+| 2
```

### 2.5.4 Exercise: Adding All The Things

#### 第一版

首先实现 `add(items: List[Int]): Int`，有两种实现方式，第一种使用 `List.foldLeft`：

```Scala
def add(items: List[Int]): Int = items.foldLeft(0)(_ + _)
```

第二种使用 `Monoid[Int]` 实例：

```Scala
def add(items: List[Int]): Int = {
  import cats.Monoid
  import cats.instances.int._
  import cats.syntax.monoid._

  items.foldLeft(Monoid[Int].empty)(_ |+| _)
}
```

#### 第二版

需要变化非常快，现在要求 `add` 函数也能实现 `Option[Int]` 的加法，即 `add` 函数需要兼容 `Int` 和 `Option[Int]` 类型入参，此时只能使用 `Monoid` 来实现了：

```Scala
import cats.Monoid
import cats.syntax.semigroup._

def add[A](items: List[A])(implicit m: Monoid[A]): A =
  items.foldLeft(m.empty)(_ |+| _)
```

此时可以通过 Scala 的 [context bound](https://docs.scala-lang.org/tutorials/FAQ/context-bounds.html) 来简化 `add` 的定义，使其更加精简：

```Scala
import cats.Monoid
import cats.syntax.semigroup._

def add[A: Monoid](items: List[A]): A =
  items.foldLeft(Monoid[A].empty)(_ |+| _)
```

* 去掉了 `(implicit m: Monoid[A])` 参数，增加 `[A: Monoid]`，`[A: Monoid]` 用来描述 `implicit` 值，表示在 `add` 函数中，有一个可用的 `Monoid[A]` 值；
* `add` 内部，通过 `Monoid[A]`（即 `Monoid.apply[A]`）来获取 `implicit` 值；

现在，只要在 `implicit` 作用域中有合适的 `Monoid` 实例，就可以使用 `add` 函数了：

```Scala
val xs = List(1, 2, 3)

import cats.instances.int._
add(xs)

val os = xs.map(Option(_))

import cats.instances.option._
add(os)
```

#### 第三版

需求继续变化，现在 `add` 需要实现对 `Order` 类型的累加：

```Scala
case class Order(totalCost: Double, quantity: Double)
```

因为 `add` 目前依赖 `Monoid` 实例实现累加操作，所以不管什么类型，只要提供该类型的 `Monoid` 实例，`add` 就可以工作：

```Scala
object Order {
  implicit val orderMonoid: Monoid[Order] =
    new Monoid[Order] {
      override def empty: Order = Order(0, 0)
      override def combine(x: Order, y: Order): Order =
        Order(x.totalCost + y.totalCost, x.quantity + y.quantity)
    }
}
```

将 `orderMonoid` 放在 `Order` 伴生对象中，省去 `import` 的麻烦，直接使用 `add` 即可：

```Scala
val orders = List(Order(1, 2), Order(3, 4))
add(orders)
```
