# 10.1 Sketching the Library Structure

上手写代码之前，先捋一捋前面的设计目标。

## Providing error messages

失败时附加有用的错误信息，则校验的结果有两种情况：

1. 返回被校验的值本身
2. 返回错误提示信息

可以用 a value in a context 来抽象，该 context 为 `the context is the possibility of an error message`：

![img](../images/a-validation-result.png)

