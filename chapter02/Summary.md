# 2.7 Summary

本章进步巨大，因为我们学会了两个 fp 中的重要抽象：

* `Semigroup`
  + 代表 `addition` 或者 `combination` 操作
* `Monoid`
  + 继承 `Semigroup`
  + 增加 **单位元** 的概念
  
在 Cats 中，导入对应的 type class, instances 和 syntax 即可使用：

```Scala
import cats.Monoid
import cats.instances.int._
import cats.syntax.semigroup._  // for |+| operator
```

只要作用域中有合适的 type class instances，就可以使用 `Monoid` 实现 `addition` 操作：

```Scala
import cats.Monoid
import cats.syntax.semigroup._

import cats.instances.set._
Set(1, 2, 3) |+| Set(1, 2, 4)

import cats.instances.map._
Map("a" → 1) |+| Map("b" → 2)

import cats.instances.string._
"Hello" |+| " world!"

import cats.instances.int._
1 |+| 2
```

通过 `Monoid` 可以实现 **非常非常** 通用的函数，例如：

```
def add[A: Monoid](items: List[A]): A =
  items.foldLeft(Monoid[A].empty)(_ |+| _)
```

只要提供合适的 `Monoid` 实例，就能实现 **任意类型** 的累加。
