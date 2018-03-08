# 3.3 Definition of a Functor

前面看到的例子，如 `List` `Option` `Future` `Function` 等都是 functor，他们都封装了 **sequencing computations**。

前面说 functor 就是具有 `map` 函数的任意类型，这当然太随意了，更正式的定义为，一个 functor 由两部分组成：

1. 一个 `F[A]`
2. 一个 `map` 函数，类型为 `(A => B) => F[B]`

因此 functor 可以实现从 `F[A]` 到 `F[B]` 的转换，而且会保持 `context`（即 `F`）不变。

其简单定义如下：

```Scala
import scala.language.higherKinds

trait Functor[F[_]]{
  def map[A, B](fa: F[A])(f: A ⇒ B): F[B]
}
```

* `F[_]` 是 type constructor，必须 `import scala.language.higherKinds` 后才能使用；

## Functor 法则

functor 可以保证以下两种操作语义相同：

1. 将很多操作一次 `map`
2. `map` 之前，将所有操作 **组合** 为一个函数，再 `map`

为达到以上效果，functor 需要满足 2 条法则：

### Identity

Identity: calling `map` with the **identity function** is the same as doing nothing:

```Scala
fa.map(a => a) == fa
```

### Composition

Composition: mapping with two functions `f` and `g` is the same as mapping with `f` and then mapping with `g`:

```Scala
fa.map(g(f(_))) == fa.map(f).map(g)
```
