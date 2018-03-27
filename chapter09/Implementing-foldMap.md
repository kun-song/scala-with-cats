# 9.2 Implementing foldMap

`Foldable.foldMap` 是基于 `foldLeft` 和 `foldRight` 的众多衍生函数中的一个。本节我们手动实现 `foldMap`，因为这有利于加深对 map-reduce 的理解。

参考下图实现 **单线程** 版 `foldMap`：

![img](../images/foldMap-algorithm.png)

```Scala
import cats.Monoid
import cats.syntax.monoid._

def foldMap1[A, B: Monoid](xs: Vector[A])(f: A ⇒ B): B =
  xs.map(f).foldLeft(Monoid[B].empty)(Monoid[B].combine)

def foldMap2[A, B: Monoid](xs: Vector[A])(f: A ⇒ B): B =
  xs.map(f).foldLeft(Monoid[B].empty)(_ |+| _)

def foldMap3[A, B: Monoid](xs: Vector[A])(f: A ⇒ B): B =
  xs.foldLeft(Monoid[B].empty)(_ |+| f(_))
```
* `B` 必须有 `Monoid` 实例
* `foldMap3` 将 map + reduce 用 `foldLeft` 同时实现

使用如下：

```Scala
import cats.instances.int._
import cats.instances.string._

foldMap1(Vector(1, 2, 3))(_ * 2)             // 12
foldMap1(Vector(1, 2, 3))(_.toString + "!")  // 1!2!3!
```

* 使用时要导入合适的 `Monoid` 实例
* `foldMap` 的行为由 `Monoid.combine` 确定，例如 `Monoid[Int]` 和 `Monoid[String]`