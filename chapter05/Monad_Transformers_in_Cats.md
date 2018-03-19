# 5.3 Monad Transformers in Cats

每个 monad transformer 都是一个数据类型，定义在 `cats.data` 中，借助 monad transformer 可以将 **嵌套** 的 monad 变成单个 monad。

要使用 monad transformer，必须理解：

1. the available transformer classes;
2. how to build **stacks of monads** using transformers;
3. how to construct instances of a **monad stack**; and
4. how to pull apart a stack to access the wrapped monads.

## 5.3.1 The Monad Transformer Classes

在 Cats 中，对 monad `Foo`，通常对应有 monad transformer `FooT`。实际上，Cats 中很多 Monad 其实是通过组合 monad transformer 和 `Id` monad 生成的。

一些常见的 monad 和其 monad transformer 如下：

* `cats.data.OptionT` for `Option`;
* `cats.data.EitherT` for `Either`;
* `cats.data.ReaderT` for `Reader`;
* `cats.data.WriterT` for `Writer`;
* `cats.data.StateT` for `State`;
* `cats.data.IdT` for the `Id` monad.

>Kleisli Arrows
>
>In Section 4.8 we mentioned that the `Reader` monad was a specialisation of a more general concept called a “kleisli arrow”, represented in Cats as `cats.data.Kleisli`.
>
>We can now reveal that `Kleisli` and `ReaderT` are, in fact, the same thing! `ReaderT` is actually a type alias for `Kleisli`. Hence why we were creating `Reader`s last chapter and seeing `Kleisli`s on the console.

## 5.3.2 Building Monad Stacks

所有 monad transformer 遵循 **相同套路**：

1. transformer 本身： monad stack 中的 **inner monad**
2. transformer 第一个类型参数：代表 monad stack 中的 **outer monad**
3. transformer 第二个类型参数：最深层的 monad context 类型

例如前面例子中的 `ListOption`，它是 `OptionT[List, A]` 的别名，而 `OptionT[List, A]` 实际上是 `List[Option[A]`，即创建 monad stack 时，采用 **由内而外** 的顺序构建：

```Scala
type ListOption[A] = OptionT[List, A]
```
