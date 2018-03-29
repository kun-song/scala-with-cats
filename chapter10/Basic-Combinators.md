# 10.3 Basic Combinators

首先为 `Check` 添加 `and` 组合子函数。

`and` 组合子接受两个 `Check`，只有两个 `Check` 都成功时，`and` 组合出的 `Check` 才会成功。

```Scala
trait Check[E, A] {
  def apply(value: A): Either[E, A]
  
  def and(that: Check[E, A]): Check[E, A] = ???
}
```

### 组合 `E`

当 `and` 的两个 `Check` **都失败** 时，需要将两者的错误信息 **一起** 返回，但目前没有什么方式能组合 `E` 的值，我们想要的效果如下：

![img](../images/combining-error-messages.svg)

* 对于类型 `E` 的两个值，有函数 `.` 将它们组合为一个 `E`

只要有 `Semigroup[E]` 实例，即可用 `combine` 或 `|+|` 语法来组合 `E` 的值：

```Scala
import cats.{Monoid, Semigroup}
import cats.instances.list._    // Semigroup[List]
import cats.syntax.semigroup._  // |+|

val listSemigroup: Semigroup[List[Int]] = Semigroup[List[Int]]

// List[Int] = List(1, 2, 3, 4, 5, 6)
listSemigroup.combine(List(1, 2, 3), List(4, 5, 6))

// List[Int] = List(1, 2, 3, 4, 5, 6)
List(1, 2, 3) |+| List(4, 5, 6)
```

>**注意**：因为不需要 **单位元**，所以无需用 `Monoid`，只用 `Semigroup` 即可（尽量使用 **更小** 的约束）。

### `and` 是否应该短路

`and` 第一个 check 失败时，是否应该采取短路策略，不再计算第二个 check 了呢？

作为校验系统，我们希望一次检查暴露尽量多错误，应尽量避免 short-circuiting，而 `and` 计算的两个 `Check` 是互相独立的，因此可以将它们一起计算完。

### 第一版实现
