# 1.5 Example: Eq

Scala 内置的 `==` 可用于 **所有对象**，因此 Scala 新手可能会写出如下代码：

```Scala
List(1, 2, 3).map(Option(_)).filter(item ⇒ item == 1)
```

这段代码的结果为 `Nil`，因为 `filter` 的断言是将 `Option[Int]` 与 `Int` 进行比较，为 `false`。

这个例子非常简单，但已经把问题表达清楚了，即 Scala 内置的 `==` 不是类型安全的，对于 `Option[Int] == Int` 这类明显错误的表达式，Scala 编译器也能编译通过（IDE 通常会给出警告），`cats.Eq` 提供了 **类型安全** 的 *equality checks*，某种程度上解决了该问题。

## 1.5.1 Equality, Liberty, and Fraternity

通过 `Eq`，可以为 *任意类型** 实现类型安全的相等比较：

```Scala
trait Eq[@sp A] extends Any with Serializable { self =>

  /**
   * Returns `true` if `x` and `y` are equivalent, `false` otherwise.
   */
  def eqv(x: A, y: A): Boolean

  /**
   * Returns `false` if `x` and `y` are equivalent, `true` otherwise.
   */
  def neqv(x: A, y: A): Boolean = !eqv(x, y)
}
```

`Eq` 的 interface syntax 定义如下：

```Scala
trait EqSyntax {
  /** not final so it can be disabled in favor of scalactic equality in tests */
  implicit def catsSyntaxEq[A: Eq](a: A): EqOps[A] =
    new EqOps[A](a)
}

final class EqOps[A: Eq](lhs: A) {
  def ===(rhs: A): Boolean = macro Ops.binop[A, Boolean]
  def =!=(rhs: A): Boolean = macro Ops.binop[A, Boolean]
}
```

其中定义了两个函数用于相等性比较：`===` 和 `=!=`，当然使用的前提是 `implicit` 作用域中存在 `Eq` instances。

## 1.5.2 Comparing Ints

可以通过 `Eq` 为 `Int` 实现类型安全的比较，首先，获取 `Eq[Int]` 实例，前面讲到每个 type class 的伴生对象中都定义了 `apply` 方法，用以获取特定类型的 type class instances，可以直接省略 `apply`：

```Scala
import cats.Eq
import cats.instances.int._

val eqInt: Eq[Int] = Eq[Int]
```

现在，可以使用 `eqInt` 对两个 `Int` 值进行比较了：

```Scala
eqInt.eqv(1, 2)
```

与 Scala 原生 `==` 不同，若用 `eqInt` 比较两个不同类型的值，会报编译错误：

```Scala
eqInt.eqv(1, "2")  // 编译报错
```

可以通过导入 interface syntax 简化语法：

```Scala
import cats.Eq
import cats.instances.int._
import cats.syntax.eq._

val eqInt: Eq[Int] = Eq[Int]

1 === 1
1 =!= 2
```

## 1.5.3 Comparing Options

现在来为 `Option[Int]` 实现 `Eq` 实例，因为 `Option[Int]` 由两个类型组成，所以要为 `Option` 和 `Int` 都导入默认实例：

```Scala
import cats.instances.int._
import cats.instances.option._
import cats.syntax.eq._

Some(1) === None
```

上面的 `Some(1) === None` 会报编译错误：

```Scala
Error:(10, 76) value === is not a member of Some[Int]
def get$$instance$$res1 = /* ###worksheet### generated $$end$$ */ Some(1) === None
                                                                          ^
```

因为目前 `implicit` 作用域中只有 `Eq[Int]` 和 `Eq[Option[Int]]` 实例，并没有 `Eq[Some[Int]]` 实例，可以通过类型转换修复该问题：

```Scala
(Some(1): Option[Int]) === None
```

更好一些的办法是，通过 `Option.apply` 和 `Option.empty` 标准库方法：

```Scala
Option(1) === None
```

或者通过 `cats.syntax.option._` 中定义的特殊语法：

```Scala
1.some === None
// or
1.some === none[Int]
```

## 1.5.4 Comparing Custom Type

通过 `Eq.instance` 辅助方法，可以自定义 `Eq` 实例，该方法参数为 `(A, A) => Boolean`：

```Scala
import java.util.Date
import java.util.concurrent.TimeUnit

import cats.Eq
import cats.instances.long._
import cats.syntax.eq._

implicit val eqDate: Eq[Date] =
  Eq.instance((d1, d2) ⇒ d1.getTime === d2.getTime)

val d1 = new Date()

TimeUnit.SECONDS.sleep(1)

val d2 = new Date()

d1 === d2  // false
d1 === d1  // true
```

* `d1.getTime === d2.getTime` 利用 `Eq[Long]` 实现

`Eq.instance` 函数的实现非常简单：

```Scala
/**
  * Create an `Eq` instance from an `eqv` implementation.
  */
def instance[A](f: (A, A) => Boolean): Eq[A] =
  new Eq[A] {
    def eqv(x: A, y: A) = f(x, y)
  }
```

## 1.5.5 Exercise: Equality, Liberty, and Felinity

### 问题

为 `Cat` 实现 `Eq[Cat]` 实例：

```Scala
final case class Cat(name: String, age: Int, color: String)
```

并使用 `Eq[Cat]` 实现比较如下两对值：

```Scala
val cat1 = Cat("Garfield",   38, "orange and black")
val cat2 = Cat("Heathcliff", 33, "orange and black")

val optionCat1 = Option(cat1)
val optionCat2 = Option.empty[Cat]
```

### 答案

`Eq[Cat]` 实例，`Cat` 的比较依赖 `Eq[Int]` 和 `Eq[String]`，所以需要导入它们的 `Eq` 实例：

```Scala
import cats.Eq
import cats.instances.int._
import cats.instances.string._
import cats.syntax.eq._

implicit val eqCat: Eq[Cat] =
  Eq.instance((cat1, cat2) ⇒ {
    cat1.name === cat2.name && cat1.age === cat2.age && cat1.color === cat2.color
  })
```

比较：

```Scala
val cat1 = Cat("Garfield",   38, "orange and black")
val cat2 = Cat("Heathcliff", 33, "orange and black")

cat1 === cat2
cat1 =!= cat2

val optionCat1 = Option(cat1)
val optionCat2 = Option.empty[Cat]

import cats.instances.option._

optionCat1 === optionCat2
optionCat1 =!= optionCat2
```

* 注意：`Option[Cat]` 使用 `===` 和 `=!=` 需要导入 `import cats.instances.option._`ß