# 2.6 Applications of Monoids

monoid 就是一个 `addition` 或者 `combination` 的抽象，那 monoid 到底有什么用呢？

## Big Data

Spark、Hadoop 等系统都会涉及对巨大数据集的操作，这些操作一般都可以抽象为 `Monoid`，参考 twitter 的 algebird 和 summingbird，在 Map-Reduce 案例中将详细介绍。

## Distributed Systems

分布式系统中的 **最终一致性** 可以通过 `Monoid` 实现，在 CRDT 案例中讲详细介绍。

## Monoids in the Small

除了在系统设计方面，`Monoid` 在小块代码方面也很有用，在第二部分的案例中有很多这种例子。
