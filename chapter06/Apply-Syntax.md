# 6.2 Apply Syntax

Cats 通过 `cats.syntax.apply` 为 6.1 节中的方法提供了方便的 apply syntax。

### `tupled`

`tupled` 简化了 `Semigroupal` 伴生对象中的 `tuple2` -> `tuple22` ，`tupled` 需要一个 `implicit` 的 `Semigroupal` 实例：

```Scala
import cats.instances.option._
import cats.syntax.apply._

// Option[(Int, Int, Int)] = Some((1,2,3))
val x = (Option(1), Option(2), Option(3)).tupled
```
* 通过 `cats.syntax.apply`，隐式地为 `tuple` 添加了 `tupled` 函数，该函数内部使用 `Semigroupal[Option]` 实例实现
* `tupled` 是通过 `Semigroupal` 实例实现的，因此只支持 2-22 数量的 `tuple`

## `mapN`

`mapN` 简化了 `Semigroupal` 伴生对象中的 `map2` -> `map22`，`mapN` 需要一个 `implicit` 的 `Functor` 实例：

```Scala
import cats.instances.option._
import cats.syntax.apply._

case class Cat(name: String, born: Int, color: String)

// Option[Cat] = Some(Cat(Mike,12,yellow))
(Option("Mike"), Option(12), Option("yellow")).mapN(Cat.apply)
```
* `mapN` 内部通过 `Semigroupal` 提取 content，然后通过 `Functor` 应用函数

`mapN` 是类型安全的函数，即 `f` 的参数类型、个数必须与 `tuple` 保持一致：

```Scala
val add: (Int, Int) => Int = (a, b) => a + b
// add: (Int, Int) => Int = <function2>

(Option(1), Option(2), Option(3)).mapN(add)
// <console>:27: error: type mismatch;
//  found   : (Int, Int) => Int
//  required: (Int, Int, Int) => ?
//        (Option(1), Option(2), Option(3)).mapN(add)
//                                               ^

(Option("cats"), Option(true)).mapN(add)
// <console>:27: error: type mismatch;
//  found   : (Int, Int) => Int
//  required: (String, Boolean) => ?z
//        (Option("cats"), Option(true)).mapN(add)
//  
``` 
