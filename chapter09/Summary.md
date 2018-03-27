# 9.4 Summary

本章实现了一个分布式的 map-reduce 系统：

1. 将数据分散到不同节点上
2. 每个结点计算自己的 map-reduce 任务
3. 将各个结点的结果进行 reduce，获取最终结果

这种依托 `Monoid` 实现的 map-reduce 并非只能计算整数相加、字符串拼接，实际上大多数数据科学家平时做的分析都可以抽象为 `Monoid`，从而可以实现这种 map-reduce 实现：

* approximate sets such as the Bloom filter;
* set cardinality estimators, such as the HyperLogLog algorithm;
* vectors and vector operations like stochastic gradient descent;
* quantile estimators such as the t-digest