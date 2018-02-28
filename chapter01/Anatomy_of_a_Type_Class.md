# 1.1 Anatomy of a Type Class

type class pattern 有 3 个主要的组成部分：

* type class 本身
* instance of type class
* 使用 type class 的 interface methods

## The Type Class

type class 是一个接口或者 API，代表我们想要实现的功能，在 Cats 中，我们使用具备至少一个类型参数的 `trait` 来表示 type class。

例如，对于“序列化为 JSON”这个功能，可以将其定义为一个 type class，首先需要定义表示 JSON 的数据结构：

```Scala
sealed trait Json

final case class JsObject(get: Map[String, Json]) extends Json
final case class JsString(get: String) extends Json
final case class JsNumber(get: Double) extends Json
case object JsNull extends Json
```

现在，可以将 “序列化为 JSON” 的功能定义为一个 type class，即 `trait`：

```Scala
trait JsonWriter[A] {
  def write(value: A): Json
}
```

## Type Class Instances

type class 的实例提供了针对 **特定类型** 的 type class 实现，因为我们使用具备类型参数的 `trait` 表示 type class，所以它的实现都是针对特定的类型参数的。

在 Scala 中，type class instance 为：使用 `implicit` 修饰的 type class(`trait`) 实例：

```Scala
// test class
final case class Person(name: String, email: String)

/**
  * Type Class Instance 即 标有 implicit 的 Type Class 实现
  */
object JsonWriterInstances {
  // type class instance 1
  implicit val stringWriter: JsonWriter[String] =
    new JsonWriter[String] {
      override def write(value: String): Json = JsString(value)
    }

  // type class instance 2
  implicit val personWriter: JsonWriter[Person] =
    person ⇒ JsObject(Map("name" → JsString(person.name), "email" → JsString(person.email)))

}
```

上面的 `stringWriter` 和 `personWriter` 都是 type class instance。

## Type Class Interfaces

type class interface 是暴露给用户的具体功能，interface 是指接受 type class instance 作为 `implicit` 参数的泛型方法。

Scala 中有两种常见的创建 interface 的方式：

* Interface Objects
* Interface Syntax

### Interface Objects

最简单的创建 interface 的方法是，将方法直接放到单例对象中：

```Scala
object Json {
  def toJson[A](value: A)(implicit writer: JsonWriter[A]): Json = writer.write(value)
}
```

要使用 `toJson`，只要导入所需 type class instance 即可，在本例中，type class instance 有 `stringWriter` 和 `personWriter` 两个：

```Scala
object Test extends App {
  // 导入 type class instance
  import JsonWriterInstances._

  val p = Person("songkun", "12345@qq.com");

  // 调用 type class interface
  val pJson = Json.toJson(p)

  // JsObject(Map(name -> JsString(songkun), email -> JsString(12345@qq.com)))
  println(pJson)
}
```

编译器在执行上面的 `Json.toJson(p)` 时，发现没有给 `toJson` 提供 `implicit` 参数，虽然少了个参数，但此时编译器没有放弃，它会去查找类型为 `JsonWriter[Person]` 的 `implicit` 值，结果找到了 `TypeClassInstances.personWriter`，编译器将其自动插入 `toJson` 方法中：

```Scala
val pJson = Json.toJson(p)(personWriter)
```

>使用 Interface Object 仅需两步：
>
>1. 导入 type class instances
>2. 调用 type class interaface

### Interface Syntax

另一种创建 interface 的方式是，利用 *方法扩展*，扩展已有类型，使其本身即具备 interface method：

```Scala
object JsonSyntax {
  implicit class JsonWriterOps[A](value: A) {
    def toJson(implicit writer: JsonWriter[A]) = writer.write(value)
  }
}
```

这里 `JsonWriterOps` 是一个 `implicit` 类，可以用来做类型转换，即将 `A` 类型的实例转换为 `JsonWriterOps` 类型，要使用 `JsonWriterOps`，我们需要一起导入 type class instances 和 `JsonWriterOps`：

```Scala
object Test extends App {
  import JsonWriterInstances._
  import JsonSyntax.JsonWriterOps

  val p = Person("songkun", "12345@qq.com");

  val pJson = p.toJson

  println(pJson)
  
}
```

这里调用 `Person.toJson`，背后发生了两件事情：

1. 编译器发现 `Person` 类没有定义 `toJson` 方法，于是查找可以通过接受一个 `Person` 参数，最终产生一个具备 `toJson` 方法的 `implicit` 类，最后找到了 `JsonWriterOps`，并将 `Person` 实例转化为 `JsonWriterOps` 实例；
2. 转换为 `JsonWriterOps` 实例后，虽然可以调用 `toJson` 方法了，但是调用时缺少了一个 `implicit` 参数，于是再次查找可用的 `implicit` 值，并自动插入调用；

### The `implicitly` Method

Scala 标准库定义了一个泛型 type class interface，名为 `implicitly`，其定义非常简单：

```Scala
@inline def implicitly[T](implicit e: T) = e
```

`implicitly` 方法接受一个 `implicit` 参数，并直接返回该值，可以使用该方法确定 `implicit` 作用域中的值是否存在，例如上面的例子中可以添加 `implicitly` 方法：

```Scala
import JsonWriterInstances._

implicitly[JsonWriter[String]]
implicitly[JsonWriter[Person]]
```

在实际使用 `JsonWriter[String]` 和 `JsonWriter[Person]` 之前，使用 `implicitly` 保证他们在隐式作用域中确实存在，如果它们都存在，则程序向下执行，若它们不存在，则编译阶段即会报错，例如：

```Scala
import JsonWriterInstances._

implicitly[JsonWriter[Int]]
```
* 无法通过编译，因为隐式作用域中不存在 `JsonWriter[Int]`！

**注意**：

我们可以在正常的代码流程中插入 `implicitly` 方法，以保证特定 type class instance 存在，且无歧义。
