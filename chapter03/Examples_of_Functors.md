# 3.1 Examples of Functors

非正式的讲，任何具备 `map` 函数的数据结构都是 functor，例如 `List` `Option` `Either` 等。

`map` 用来 **转换** functor 中的所有值，而且是在 *一次* 遍历中完成的，`map` 不会影响 context，即原来是 `List`，转换后依然还是 `List`：

```Scala
List(1, 2, 3).map(n ⇒ n + 1)
```

当 `map` 作用于某数据结构时，`map` 仅仅转换其 **内容**，而不改变 **context**：

* 作用于 `Option` 时，原来是 `Some`，转换后也是 `Some`，原来是 `None`，转换后也是 `None`
* 作用域 `Either` 时，原来是 `Left`，转换后也是 `Left`，原来是 `Right`，转换后也是 `Right`
