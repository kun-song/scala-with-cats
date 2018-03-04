# 1.4 Meet Cats

前面讲解了如何在 Scala 中实现 type class，本节介绍 Cats 是如何实现 type class 的。

Cats 采用 **模块化** 的结构，允许使用者导入所需的 type class，instances 和 interface method。

`cats.Show` 类型 1.3 练习中定义的 `Printable`，可以在不使用 `toString` 条件下，提供开发者友好的标准终端输出功能，其简单定义如下：

```Scala
trait Show[T] {
  def show(t: T): String
}
```

## Importing Type Classes

Cats 中，type class 被定义在 `cats` 包中，可以直接导入：

```Scala
import cats.Show
```

每个 Cats 中的 type class 的 **伴生对象** 中都有一个 `apply` 方法，用来定位特定类型的 type class instance，类似 Scala 原生的 `implicitly` 函数。

例如，要获取 `Show[Int]` 可以：

```Scala
object ShowDemo extends App {
  val showInt = Show.apply[Int]
}
```

但编译后发现，以上代码会抛出：

```Scala
Error:(11, 27) could not find implicit value for parameter instance: cats.Show[Int]
  val showInt = Show.apply[Int]
```

查看 `apply` 函数的实现：

```Scala
def apply[A](implicit instance: Show[A]): Show[A] = instance
```

`apply` 需要一个 `implicit` 的 type class instance，然后直接返回 instance 本身，因此要使用 `apply`，`implicit` 作用域中必须有对应的 type class instance，本例中，即为 `Show[Int]`。

## Importing Default Instances

`cats.instances` 中包含很多类型的 default instances，可以通过如下语句导入它们：

```Scala
cats.instances.int
cats.instances.string
cats.instances.list
cats.instances.option

cats.instances.all
```

其中，`cats.instances.option` 等包含了所有针对 `Option` 类型的 type class instances，而 `cats.instances.all` 则包含了所有 Cats 提供的 type class instances。

现在，导入针对 `String` `Int` 类型的 `Show` 实例：

```Scala
object ShowDemo extends App {
  import cats.instances.int._
  import cats.instances.string._

  val showInt: Show[Int] = Show.apply[Int]
  val showString: Show[String] = Show.apply[String]
}
```

注意，`import cats.instances.int._` 后，`Show.apply[Int]` 不再报错，我们可以按如下方式使用 `showInt` 和 `showString`：

```Scala
val intAsString: String = showInt.show(20)
val stringAsString: String = showString.show("abc")
```

## Importing Interface Syntax

类似 `Printable.format(cat)` 可以使用 `cat.format` 替代一样，可以通过从 `cats.syntax.show` 导入 *interface syntax* 来使 `Show` type class 更容易使用：

```Scala
import cats.syntax.show._

val intAsString = 20.show
val stringAsString = "abc".show
```

## Importing All the Things!

* `import cats._` 导入全部 type class
* `import cats.instances.all._` 导入全部 type class instances
* `import cats.syntax.all._` 导入全部 syntax
* `import cats.implicits._` 导入全部 type class instances 和 syntax

所以，如果要导入全部 type class + type class instances + syntax，可以：

```Scala
import cats._
import cats.implicits._
```

## Defining Custom Instances

使用 Cats 可以很容易自定义 type class instances，只要实现 type class 对应的 `trait` 即可，例如：

```Scala
import java.util.Date

import cats._

implicit val dataShow: Show[Date] =
  date => s"${date.getTime}  ms since the epoch."
```

使用如下：

```Scala
import cats.implicits._

new Date().show
```

上面例子中的 `dataShow` 是通过 *single abstract method* 创建的，它本质是 `new Show[Data] {...}` 的简写，虽然简化了一些，但本质上还是从头到尾实现了一个 `trait`，Cats 在 `Show` 的伴生对象中提供了两个辅助方法，用于简化 `Show` instances 的创建过程：

```Scala
object Show {
    /** creates an instance of [[Show]] using the provided function */
  def show[A](f: A => String): Show[A] = new Show[A] {
    def show(a: A): String = f(a)
  }

  /** creates an instance of [[Show]] using object toString */
  def fromToString[A]: Show[A] = new Show[A] {
    def show(a: A): String = a.toString
  }
}
```
* `show` 可以从函数 `A => String` 创建 `Show[A]`
* `fromToString` 可以从 `toString` 创建 `Show[A]`

借助着两个函数，可以使创建 `Show` 实例的过程更加简单、灵活：

```Scala
implicit val dataShow: Show[Date] = 
  Show.show(date ⇒ s"${date.getTime} ms since the epoch.")
```

Cats 中的很多 type class 都提供了类似 `Show.show` 这样的函数，用于简化创建 type class instances 的过程。

## Exercise: Cat Show

使用 `Show` 替代 `Printable`，重新实现 1.3 节中的练习。

### 第一步：定义 type class

因为 `Show` 已经在 Cats 中实现，所以第一步省略。

### 第二步：使用 type class

定义 `Cat` 类：

```Scala
case class Cat(name: String, age: Int, color: String)
```

为 `Cat` 定义 `Show[Cat]`：

```Scala
object Cat {
  import cats.Show
  import cats.instances.string._
  import cats.instances.int._
  import cats.syntax.show._

  implicit val catShow: Show[Cat] =
    cat ⇒ {
      val name = cat.name.show
      val age = cat.age.show
      val color = cat.color.show

      s"$name is a $age year-old $color cat."
    }
}
```

因为 `Show` 的interface method 并没有定义在 `Show` 的伴生对象中，所以最直接的使用方式为直接调用 `Cat` 伴生对象中的 `catShow` 实例：

```Scala
Cat.catShow.show(Cat("Mike", 7, "yellow"))
```

### 第三步：更友好的语法

导入 `Show` type class 的 interface syntax 即可：

```Scala
import cats.syntax.show._
// or
//import cats.implicits._

Cat("Mike", 7, "yellow").show
```