# 9.1 Parallelizing map and fold

回忆 `Functor.map` 的语义，它将函数 `A => B` 应用到 context `F` 中的值上，即：

![img](../images/functor-map-type.png)

`map` 独立的对 sequence 中的元素进行转换，因为 **独立**，所以 `map` 很容易并行化。

至于 `fold`，完全可以用 `Foldable` 实现，问题是并非所有 `Functor` 同时也是 `Foldable`，但如果类型 `A` 同时满足 `Functor` 和 `Foldable`，则可以在 `A` 之上创建 map-reduce 系统。

有了 `fold`，则 reduce 阶段变成了对 **分布式计算的 `map` 结果** 做 `foldLeft`，`map` 是独立的，并行化没有问题，但 `fold` 并行化后丧失了对 sequence 元素的 **遍历顺序**，因此要求 `foldLeft` 的 binary function 满足 **结合律**，从而消除丧失顺序带来的影响：

```Scala
reduce(a1, reduce(a2, a3)) == reduce(reduce(a1, a2), a3)
```

`fold` 函数还需要提供一个 accumulator，因为 `fold` 已经并行化，所以要求初始 accumulator 不影响计算，即要求它必须是 **identity element**：

```Scala
reduce(seed, a1) == reduce(a1, seed) == a1
```

`fold` 要求类型 `B` 满足结合律，且具备单位元，一切回到了起点，这正是本书介绍的第一个 type class `Monoid`。

实际上 `Monoid` 确实非常重要，现代很多大数据处理系统都是基于 `Monoid` 实现的，例如 twitter 的 SummingBird。