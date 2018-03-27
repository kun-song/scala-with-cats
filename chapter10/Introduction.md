# Chapter 10 Case Study: Data Validation

本章实现一个校验库，校验无处不在，例如表单输入、配置文件校验、服务响应正确性校验等等。

一些常见的校验内容如下：

1. A user must be over 18 years old or must have parental consent.
2. A `String` ID must be parsable as a `Int` and the `Int` must correspond to a valid record ID.
3. A bid in an auction must apply to one or more items and have a positive value.
4. A username must contain at least four characters and all characters must be alphanumeric.
5. An email address must contain a single `@` sign. Split the string at the `@`. The string to the left must not be empty. The string to the right must be at least three characters long and contain a dot.

从这些例子，可以总结一些 **设计目标**：

* 校验失败时，能够添加 **提示信息**
* 能够 **组合** 校验
* 校验的同时，能够 **转换** 数据
* 能一次性收集所有校验信息（accumulating error-handling）