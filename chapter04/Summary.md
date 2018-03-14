# 4.11 Summary

本章将 `flatMap` 视为一种 **排列计算（函数）** 的 **操作符**，函数必须按照 `flatMap` 指定的 **顺序** 依次执行，从这个角度看：

* `Option` represents a computation that can fail without an error message
* `Either` represents computations that can fail with a message
* `List` represents multiple possible results
* `Future` represents a computation that may produce a value at some point in the future