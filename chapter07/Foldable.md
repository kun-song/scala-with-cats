# 7.1 Foldable

Scala 标准库中很多序列（例如 `List`, `Vector`, `Stream` 等）都定义有 `foldLeft` 和 `foldRight` 函数，`Foldable` 将它们抽象出来了。

`Foldable` 还是展示 `Monoid` 和 `Eval` 使用的很好示例。

## 7.1.1 Folds and Folding

首先看下 fold 的基本用法：

```Scala
def show[A](xs: List[A]): String =
  xs.foldLeft("nil")((acc, x) ⇒ s"$x then $acc")

// String = nil
show(Nil)

// String = 3 then 2 then 1 then nil
show(1 :: 2 :: 3 :: Nil)
```
* `foldLeft` 接受一个 accumulator 和一个 binary combinator
* `foldLeft` 从左到右，依次遍历 `xs`，对 `xs` 中每个 `x` 都执行 binary combinator 计算，其结果作为新的 accumulator
* 最后，到达 `xs` 末尾时，accumulator 即为最后的计算结果

根据给定 binary function 的不同，遍历序列的 **顺序** 可能导致不同结果，因此 `fold` 有 2 个重要变体：

1. `foldLeft`
  * 从左到右遍历
2. `foldRight`
  * 从右到左遍历

上面说的从左到右和从右到左，指的是 **逻辑概念**，实际实现中，元素都是按照 **从左到右** 的顺序遍历的：

* `foldLeft` 逻辑顺序、实际顺序相同
* `foldRight` 逻辑顺序、实际顺序 **相反**，因此执行时会导致函数调用 **不断入栈**，可能导致栈溢出

可以用两幅图解释 `foldLeft` 和 `foldRight` 实际计算过程的不同：

![img](../images/foldLeft.png)

![img](../images/foldRight.png)

实际遍历顺序是自顶向下：

1. `foldLeft` 每次遍历都会执行给定的 binary function
2. `foldRight` 只有到达最底层的那个元素才开始 **第一次计算**，然后再向上依次返回、计算
  * 每层计算是一次递归调用，需要用 stack 缓存调用栈
  * 因此，如果元素过多，就会导致 **栈溢出**

若给定的 binary function 满足 **交换律**，则 `foldLeft` 和 `foldRight` 等价：

```Scala
List(1, 2, 3).foldLeft(0)(_ + _)  // 6
List(1, 2, 3).foldRight(0)(_ + _) // 6
```
* `+` 满足结合律
* 等价仅仅指两个逻辑等价，但是 `foldRight` 仍然可能 **栈溢出**！

若 binary function 不满足结合律，则 `foldLeft` 和 `foldRight` 不等价：

```Scala
List(1, 2, 3).foldLeft(0)(_ - _)  // -6

List(1, 2, 3).foldRight(0)(_ - _) // 2
```
* `-` 不满足结合律

## 7.1.2 Exercise: Reflecting on Folds

用 `Nil` 作为 accumulator，用 `::` 作为 binary function，`foldLeft` 与 `foldRight` 结果有何不同？

```Scala
val xs = 1 :: 2 :: 3 :: Nil

// List(3, 2, 1)
xs.foldLeft(List.empty[Int])((acc, x) ⇒ x :: acc)

// List(1, 2, 3)
xs.foldRight(List.empty[Int])((x, acc) ⇒ x :: acc)
xs.foldRight(List.empty[Int])(_ :: _)
```
* `::` 两边元素的顺序，导致 `foldRight` 可以用更简洁的写法

## 7.1.3 Exercise: Scaf-fold-ing Other Methods

`foldLeft` 和 `foldRight` 是非常通用的函数，使用 `foldRight` 为 `List` 实现 `map` `flatMap` `filter` `sum` 函数。

`map` `flatMap` 和 `filter` 定义非常简单，根据类型即可推导出：

```Scala
def map[A, B](xs: List[A])(f: A ⇒ B): List[B] =
  xs.foldRight(List.empty[B])((a, acc) ⇒ f(a) :: acc)

def flatMap[A, B](xs: List[A])(f: A ⇒ List[B]): List[B] =
  xs.foldRight(List.empty[B])((a, acc) ⇒ f(a) ++ acc)

def filter[A](xs: List[A])(p: A ⇒ Boolean): List[A] =
  xs.foldRight(List.empty[A]) { (a, acc) ⇒
    if (p(a)) a :: acc
    else acc
  }
```

但 `sum` 没有那么直观，`sum` 签名如下：

```Scala
def sum[A: Monoid](xs: List[A]): A
```

`A` 是一个类型参数，不知道它的具体类型，怎么可能对 `A` 求和呢？

回想下 `Monoid` 的定义，`Monoid` 具有 `empty` 和 `combine`，天然适合用于 `fold`：

```Scala
def sum[A: Monoid](xs: List[A]): A =
  xs.foldRight(Monoid[A].empty)(Monoid[A].combine)
```

## 7.1.4 Foldable in Cats

Cats 将 `foldLeft` 和 `foldRight` 抽象为 `Foldable` type class，因此所有 `Foldable` 实例都有这两个函数，且还会继承一系列它们的衍生函数，Cats 为 `List` `Vector` `Option` `Stream` 等很多 Scala 原生类型定了 `Foldable` 实例。

像以前一样，可以用 `Foldable.apply` 取得当前作用域中的 `Foldable` 实例：

```Scala
import cats.Foldable
import cats.instances.list._  // for Foldable[List] instance

val xs = 1 :: 2 :: 3 :: Nil

Foldable[List].foldLeft(xs, 0)(_ + _)  // Int = 6
```

其他 sequence，例如 `Vector` `Stream` 用法类似 `List`，而 `Option` 被视为有 0 个或者 1 个元素的 sequence：

```Scala
import cats.Foldable
import cats.instances.option._  // for Foldable[Option] instance
import cats.syntax.option._

Foldable[Option].foldLeft(111.some, 6)(_ * _)  // Int = 666
Foldable[Option].foldLeft(none[Int], 6)(_ * _)  // Int = 6
```

### 7.1.4.1 Folding Right

`Foldable` 对 `foldRight` 定义与 `foldLeft` 不同，有些特别：

```Scala
def foldRight[A, B](fa: F[A], lb: Eval[B])(f: (A, Eval[B]) => Eval[B]): Eval[B]
```

* `foldRight` 通过 `Eval` 实现 lazy evaluation，从而保证 stack safe；
* `foldRight` 结果为 `Eval[B]`，要获取 `B` 需要通过 `Eval.value` 强制计算；

通过使用 `Eval` 作为 accumulator，`Foldable.foldRight` 是栈安全的，与此相对，某些 Scala 集合类原生的 `foldRight` 就不是栈安全的，例如 `Stream`：

```Scala
import cats.Eval
import cats.Foldable

def bigData = (1 to 100000).toStream

bigData.foldRight(0L)(_ + _)
// java.lang.StackOverflowError ...
```
* 抛出 `StackOverflowError`

通过使用 `Foldable[Stream].foldRight` 可以避免栈溢出：

```Scala
import cats.{Eval, Foldable}
import cats.instances.stream._

val bigData = (1 to 100000).toStream

val sum: Eval[Int] = Foldable[Stream].foldRight(bigData, Eval.now(0))((x, lb) ⇒ lb.map(_ + x))

// Int = 705082704
sum.value
```

>Stack Safety in the Standard Library
>
>使用标准库集合时，栈安全并非要考虑的重点，因为 Scala 标准库集合中的 `foldRight` 基本都是栈安全的：
>```
>(1 to 100000).toList.foldRight(0L)(_ + _)
>// res8: Long = 5000050000
>
>(1 to 100000).toVector.foldRight(0L)(_ + _)
>// res9: Long = 5000050000
>```
>这里用 `Stream` 举例是因为它是一个例外，`Stream.foldRight` 并非栈安全的，然而我们用 `Eval` 修复了改问题，哈哈！

### 7.1.4.2 Folding with Monoids

`Foldable` 基于 `foldLeft` 实现了一系列的有用函数，它们大多数与标准库的函数对应，例如 `find` `exists` `forall` `toList` `isEmpty` `nonEmpty` 等：

```Scala
import cats.Foldable
import cats.instances.option._
import cats.instances.list._

Foldable[Option].nonEmpty(Option(111))  // true

Foldable[List].find(List(1, 2, 3))(_ % 2 == 0)  // Some(2)
```

除了与标准库对应的函数外，`Foldable` 基于 `Monoid` 定义了两个独有函数：

* `combineAll`/`fold`
  + combines all elements in the sequence using their `Monoid`
* `foldMap`
  + maps a user-supplied function over the sequence and combines the results using a `Monoid`

#### `combineAll`/`fold`

例如，使用 `combineAll`/`fold` 对 `List[Int]` 求和：

```Scala
import cats.Foldable
import cats.instances.int._  // for Monoid[Int]
import cats.instances.list._  // for Foldable[List]

val xs = 1 :: 2 :: 3 :: Nil

Foldable[List].combineAll(xs)  // 6
Foldable[List].fold(xs)        // 6
```

* 使用 `Foldable[List].combineAll` 对 `List` 求和
* `combineAll` 使用 `Monoid[Int].combine` 对 `Int` 求和
* `fold` 是 `combineAll` 的类型别名

#### `foldMap`

还可以用 `foldMap` 对列表元素 `map` 后再 `fold`：

```Scala
import cats.Foldable
import cats.instances.int._    // for Monoid[Int]
import cats.instances.string._ // for Monoid[String]
import cats.instances.list._   // for Foldable[List]

val xs = 1 :: 2 :: 3 :: Nil

// String = 123
Foldable[List].foldMap(xs)(_.toString)

// Int = 12
Foldable[List].foldMap(xs)(_ * 2)
```

#### `compose`

使用 `compose` 实现 **嵌套 sequences** 的元素遍历：

```Scala
import cats.Foldable
import cats.instances.int._    // for Monoid[Int]
import cats.instances.vector._ // for Foldable[String]
import cats.instances.list._   // for Foldable[List]

val xs = List(Vector(1, 2, 3), Vector(4, 5, 6))

(Foldable[List] compose Foldable[Vector]).fold(xs)  // 21
``` 

### 7.1.4.3 Syntax for Foldable

`cats.syntax.foldable` 为 `Foldable` 中所有函数都提供了 syntax 支持，且套路相同：`Foldable` 中函数的第一个参数，变为 syntanx 函数的调用者：

```Scala
import cats.instances.int._    // for Monoid[Int]
import cats.instances.list._   // for Foldable[List]
import cats.syntax.foldable._

List(1, 2, 3).combineAll      // Int = 6
List(1, 2, 3).foldMap(_ * 2)  // Int = 12
```

>Explicits over Implicits
>
>注意，只有当调用者本身 **未定义** 该函数是，Scala 才会使用 `Foldable` 中的该函数，若调用者已经定义了，则使用它自己的那个，例如下面的 `foldLeft` 将使用 `List.foldLeft` 而非 `Foldable.foldLeft`：
>
>```
>List(1, 2, 3).foldLeft(0)(_ + _)
>```
>
>而下面将使用 `Foldable.foldLeft`：
>
>```
>import cats.Foldable
>import cats.syntax.foldable._
>
>import scala.language.higherKinds
>
>def sum[F[_]: Foldable](xs: F[Int]): Int = xs.foldLeft(0)(_ + _)
>```
>
>但这并非问题，相反这正是我们需要的：我们随意调用函数，Scala 编译器将决定何时使用 `Foldable` 才能正常工作，而如果要用 `Foldable.foldRight` 只要用 `Eval` 做为 accumulator 即可。
