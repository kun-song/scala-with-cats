# 1.3 Exercise: Printable Library

Scala 为所有类型定义了 `toString` 方法，但该方法有不少限制（具体不表），现在实现 `Printable` type class，达到更好的 `toString` 功能。

## 第一步：定义 type class

前面讲到，type class 由 3 部分组成：

* type class 本身
* instances of type class
* interface method

所以，我们分为 3 步来定义 type class：

1. 定义 type class
  + 定义 type class `Printable[A]`，包含一个方法 `format`，`format` 入参类型为 `A`，返回一个 `String`；
2. 定义 instances of type class
  + 创建 `PrintableInstances` 对象，包含针对 `String` `Int` 类型的 type class instances；
3. 定义 interface methods：
  + 定义 `Printable` 对象，包含 2 个接口方法：
    - `format` 参数为 `A` 和 `implicit Printable`，实现中使用 `Printable` 实例将 `A` 转换为 `String`，`format` 返回值为 `String`；
    - `print` 参数与 `format` 相同，但返回类型为 `Unit`，`print` 方法将 `A` 使用 `println` 打印输出；

代码清单如下：

```Scala
trait Printable[A] {
  def format(value: A): String
}

object PrintableInstances {
  implicit val stringPrintable: Printable[String] = v ⇒ v
  implicit val intPrintable: Printable[Int] = i ⇒ i.toString
}

object Printable {
  def format[A](value: A)(implicit p: Printable[A]): String =
    p.format(value)
  def print[A](value: A)(implicit p: Printable[A]): Unit =
    println(format(value))
}
```

## 第二步：使用 type class

第一步中定义的 type class 可以在很多应用中使用，一般如下使用：

首先，需要根据业务定义领域模型，例如：

```Scala
final case class Cat(name: String, age: Int, color: String)
```

接下来，需要为类型 `Cat` 定义一个 type class instance，该实现需要按 `NAME is a AGE year-old COLOR cat.` 格式输出 `Cat` 的表示：

```Scala
object Cat {
  import PrintableInstances._

  implicit val catPrintable: Printable[Cat] =
    cat ⇒ {
      val name = Printable.format(cat.name)
      val age = Printable.format(cat.age)
      val color = Printable.format(cat.color)

      s"$name is a $age year-old $color cat."
    }
}
```
* 注意 type class instance 有 4 中打包方式，这里我们将其打包到 type parameter（cat）的半生对象中，这样，使用时就无需手动 `import` 了；
* 注意 `Printable[Cat]` 的实现依赖 `stringPrintable` 和 `intPrintable`，因此需要将它们 `import` 到当前作用域；

到此为止，已经可以使用 `Printable[Cat]` 对 `Cat` 进行格式化和输出了：

```Scala
object PrintableTest extends App {
  val cat = Cat("Mike", 4, "yellow")
  
  // 1. 使用 Printable.print
  Printable.print(cat)
  // 2. 使用 Printable.foramt
  println(Printable.format(cat))
}
```

因此我们将 `catPrintable` 定义在了 `Cat` 的伴生对象中，因此不需要手动将其 `import` 到当前作用域即可使用。

>注意
>
>`catPrintable` 的定义揭示了使用 type class 的一般套路，type class 是为 **所有类型** 定义的，而使用又是针对 **特定类型** 的，比如 `Cat`，因此，首先需要针对业务领域建模的类型，定义特定的 type class instance，然后再使用 interface method。
>
>type class instance 三大组成部分中，使用者一般需要根据 type class 和已有的 type class instances 定义自己的 type class instances，然后使用 interface method。 

## 第三步：更友好的语法

第二步中，我们通过 `Printable.print` 和 `Printable.format` 两个函数使用 type class，这样有点稍微啰嗦，可以利用 *方法扩展* 实现更友好的语法.

首先，定义 `PrintableSyntax` 对象，其中包含一个名为 `PrintableOps[A]` 的 `impclicit class`： 

```Scala
object PrintableSyntax {
  implicit class PrintableOps[A](value: A) {
    def format(implicit p: Printable[A]): String = p.format(value)
    def print(implicit p: Printable[A]): Unit = println(p.format(value))
  }
}
```

有了 `PrintableSyntax` 对象，可以像使用 `Cat` 的原生方法那样使用 `print` 和 `format` 函数：

```Scala
object PrintableTest extends App {
  import PrintableSyntax._

  val cat = Cat("Mike", 4, "yellow")

  cat.print

  println(cat.format)
}
```
* 注意，此时需要手动 `import PrintableSyntax._`
