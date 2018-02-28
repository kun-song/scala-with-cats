# 1.2 Working with Implicits

在 Scala 中使用 type class，则必然会使用到 `implicit` 值和 `implicit` 参数，若想有效使用它们，有几条规则需要注意。

## Packaging Implicits

Scala 规定，所有使用 `implicit` 修饰的定义，必须放在 `object` 或 `trait` 内部，禁止放在 top-level 作用域中。

1.1 节的例子中，我们把 `implicit` type class instance 放在 `JsonWriterInstances` 对象内部，当然也可以把它们放到 `JsonWriter` 的伴生对象中，但在 Scala 中，将实例放在伴生对象中具有 *特殊作用*，这会影响到 *implicit scope*。