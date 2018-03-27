# Chapter 9 Case Study: Map-Reduce

本章将使用 `Monoid` `Functor` 等实现一个简单却强大的并行处理框架，即 MapReduce，MapReduce 由 map 和 reduce 两个步骤组成：

1. map，即 `Functor.map`
2. reduce，在 Scala 中一般叫做 `fold`
