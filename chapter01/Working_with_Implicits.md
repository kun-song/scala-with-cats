# 1.2 Working with Implicits

在 Scala 中使用 type class，则必然会使用到 `implicit` 值和 `implicit` 参数，若想有效使用它们，有几条规则需要注意。

## Packaging Implicits

Scala 规定，所有使用 `implicit` 修饰的定义，必须放在 `object` 或 `trait` 内部，禁止放在 top-level 作用域中。

1.1 节的例子中，我们把 `implicit` type class instance 放在 `JsonWriterInstances` 对象内部，当然也可以把它们放到 `JsonWriter` 的伴生对象中，但在 Scala 中，将实例放在伴生对象中具有 *特殊作用*，这会影响到 *implicit scope*。

## Implicit Scope

Scala 编译器时通过类型来查找 type class instance 的，例如 `Json.toJson("hello")`，编译器将会查找针对 `String` 类型的 type class instance，即 `JsonWriter[String]`。

编译器对 `implicit` 值的查找发生在 `toJson` 方法调用的地方，查找的范围被称为 `implicit scope`，包括：

1. 局部作用域中，或者继承而来的 **定义**
2. `import` 进来的定义
3. type class 伴生对象，或者 type class 的 type parameter 伴生对象中的定义

对前面的例子，第 3 点钟的 type class 为 `JsonWriter`，type class 的类型参数为 `String`，因次 `JsonWriter` 和 `String` 伴生对象都算作其 `implicit scope` 的一部分。

注意：

1. 只有被 `implicit` 修饰的定义才会包括在 `implicit scope` 中，普通定义不会；
2. 对每个类型，隐式作用域中只能包含一个 `implicit` 值，都则编译器无法决定到底使用哪个，产生歧义错误；

实际的 `implicit` 解析非常复杂，这里介绍解析规则，是为了理解 type class instances 的打包方式，目的已经达到，我们一般有 4 种打包 type class instances 的方法：

1. 将 type class instances 放到某个 object 中，例如 `JsonWriterInstances`；
2. 将 type class instances 放到某个 `trait` 中；
3. 将 type class instances 放到 type class 的伴生对象中；
4. 将 type class instances 放到 parameter type 的伴生对象中；

这 4 种打包方式，导入 type class instances 的方式不同：

1. 方式 1：通过 `import` 导入
2. 方式 2：通过继承导入
3. 方式 3 & 4：无需引入，它们总是在 `implicit scope` 中

## Reursive Implicit Resolution

Scala 编译器在寻找 `implicit` type class instances 时，可以 **组合** 多个 `implicit` 定义，这非常强大。

一般有 2 种定义 type class instances 的方式：

1. `implicit val`：将 instance 定义为具体的实现
2. `implicit def`：通过定义 `implicit` 方法，该方法接受 type class instance，产生另外一个 type class instance

使用 `implicit val` 定义 instance 非常直观，很容易理解，但为什么要通过 `implicit method` 将一个 instance 转换为另外一个 instance 呢？直接定义不是更直接？

这么做自然有其道理，假如要为 `Option` 定义 `JsonWriter` 实例，对于每个类型参数 `A`，我们都需要一个 `JsonWriter[Option[A]]`，当然可以暴力解决：为每个 `A` 定义 `JsonWriter[A]` 和 `JsonWriter[Option[A]]`，但这样会极大增加工作量。

其实，可以通过 `JsonWriter[A]` 来实现 `JsonWriter[Option[A]]`，原理很简单，即将 `JsonWriter[Option[A]]` 的工作代理给 `JsonWriter[A]` 即可：

```Scala
/**
 * 注意，optionWriter 的参数必须是 implicit 参数，否则 optionWriter 将不参与 implicit resolution！
 */
implicit def optionWriter[A](implicit w: JsonWriter[A]): JsonWriter[Option[A]] =
  option ⇒ option match {
    case Some(a)  ⇒ w.write(a)  // 代理
    case None     ⇒ JsNull
  }
```

`optionWriter` 方法接受一个 `implicit JsonWriter[A]`，返回结果为 `JsonWriter[Option[A]]`，从效果上看，就是将入参的 type class instance 转为返回值的 type class instance，最终结果是：**`optionWriter` 方法定义了一个 `JsonWriter[Option[A]]` type class instance**。

有了 `JsonWriter[Option[A]]`，我们可以这样写：

```Scala
Json.toJson(Option["hello"])
```

回一下 `toJson` 方法的定义：

```Scala
object Json {
  def toJson[A](value: A)(implicit writer: JsonWriter[A]): Json = writer.write(value)
}
```

在本例中，类型参数是 `Option[String]`，因此编译器将查找类型为 `JsonWriter[Option[String]]` 的 `implicit` 实例，经过查找，`implicit` 作用域中有一个 `implicit` 方法的返回值恰好是 `JsonWriter[Option[String]]`，于是编译器自动将该方法插入方法调用中：

```Scala
Json.toJson(Option("hello"))(optionWriter[String])
```

但是 `optionWriter` 毕竟是个方法，编译器将尝试执行它，以获取真正的 `toJson` 的参数，执行过程中，编译器发现它的参数也是 `implicit`，且类型为 `JsonWriter[String]`，所以再次查找 `implicit` 作用域，最后找到了 `stringWriter` 值，满足要求，再次自动插入：

```Scala
Json.toJson(Option("hello"))(optionWriter(stringWriter))
```

到此为止，`Json.toJson(Option("hello"))` 的参数已经全部就位，可以正确执行了。

从该例中可以看到，`implicit` 解析，变成了对 `implicit` 定义的组合空间的 **查找**。


>隐式转换
>
>注意，创建 `optionWriter` 时，必须将它的参数标注为 `implicit`，否则编译器在执行 implicit resolution 时，会将其排除在外。
>
>参数不是 `implicit` 的 `implicit` 方法，实际上是另一个 Scala 模式，即 implicit conversion，这是很老的编程模式，现代 Scala 已经不鼓励使用。
>
>若去掉 `optionWriter` 参数中的 `implicit` 关键字，编译器将报错，此时需要明确开启 implicit conversion 特性：`import scala.language.implicitConversions`！