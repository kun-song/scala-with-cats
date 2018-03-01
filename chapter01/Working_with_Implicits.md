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
