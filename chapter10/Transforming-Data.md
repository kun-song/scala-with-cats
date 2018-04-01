# 10.4 Transforming Data

另一个设计目标是 **数据转换**，有了转换的能力就可以对输入进行解析。

最先想到的肯定是 `map`，但我们对 `Check` 的定义要求输入与输出的 **类型相同**：

```Scala
type Check[E, A] = A => Either[E, A]
```

因此要实现 `map` 需要修改 `Check` 的类型，将其输入类型、输出类型分离：

```Scala
type Check[E, A, B] = A => Either[E, B]
```

这样一来，可以实现对输入的解析：

```Scala
val parseInt: Check[List[String], String, Int] =
```

但将输入类型、输出类型分离带来了另一个问题，即 `and` 和 `or` 实现中都假设 `Check` 成功时会将输入 **原封不动** 地返回，因此直接忽略 `Check.apply` 的计算结果，直接将 `value` 返回：

```Scala
case And(left, right) ⇒ (left(value), right(value)).mapN((_, _) ⇒ value)
                                ^             ^                     ^
```

这暗示我们对 `Check` 的抽象是错误的！

## 10.4.1 Predicates

解决方式是区分 predicat 和 check 两个概念：

* predicat 可以用逻辑操作符如 `and` `or` 组合
* check 用于 **数据转换**

将前面实现的 `Check` 重命名为 `Predicat`，且 `Predicate` 若计算成功，必定原封不动返回其输入，这可以用 identity law 来保证：

>For a predicate `p` of type `Predicate[E, A]` and elements `a1` and `a2` of type `A`, if `p(a1) == Success(a2)` then `a1 == a2`.

重构后 `Predicate` 如下：

```Scala
import cats.Semigroup
import cats.data.Validated
import cats.data.Validated.{Invalid, Valid}
import cats.syntax.validated._
import cats.syntax.semigroup._ // |+|
import cats.syntax.apply._     // mapN

sealed trait Predicate[E, A] {
  def and(that: Predicate[E, A]): Predicate[E, A] = And(this, that)

  def or(that: Predicate[E, A]): Predicate[E, A] = Or(this, that)

  def apply(value: A)(implicit s: Semigroup[E]): Validated[E, A] =
    this match {
      case Pure(f)          ⇒ f(value)
      case And(left, right) ⇒ (left(value), right(value)).mapN((_, _) ⇒ value)
      case Or(left, right)  ⇒
        left(value) match {
          case Valid(v)   ⇒ v.valid
          case Invalid(x) ⇒
            right(value) match {
              case Valid(v)   ⇒ v.valid
              case Invalid(y) ⇒ (x |+| y).invalid
            }
        }
    }
}

final case class And[E, A](left: Predicate[E, A], right: Predicate[E, A]) extends Predicate[E, A]
final case class Or[E, A](left: Predicate[E, A], right: Predicate[E, A]) extends Predicate[E, A]

final case class Pure[E, A](f: A ⇒ Validated[E, A]) extends Predicate[E, A]
```

## 10.4.2 Checks

### `map`

`Check` 应该可以从 `Predicate` 创建：

```Scala
import cats.Semigroup
import cats.data.Validated

/**
  * Author: Kyle Song
  * Date:   下午2:18 at 18/3/31
  * Email:  satansk@hotmail.com
  */
sealed trait Check[E, A, B] {
  def apply(value: A)(implicit s: Semigroup[E]): Validated[E, B]

  def map[C](f: B ⇒ C): Check[E, A, C] = Map(this, f)
}

object Check {
  def apply[E, A](p: Predicate[E, A]): Check[E, A, A] = Pure(p)
}

final case class Map[E, A, B, C](check: Check[E, A, B], f: B ⇒ C) extends Check[E, A, C] {
  def apply(value: A)(implicit s: Semigroup[E]): Validated[E, C] =
    check(value).map(f)
}

final case class Pure[E, A](p: Predicate[E, A]) extends Check[E, A, A] {
  def apply(value: A)(implicit s: Semigroup[E]): Validated[E, A] = p(value)
}
```

`map` 的语义为，给定 `Check[E, A, B]`，以及函数 `B => C`，获取 `Check[E, A, C]`，没有改变代表错误的 `E` 和代表输入的 `A`，函数 `f` 仅对输出进行修改。

### `flatMap`

`flatMap` 的语义呢，参考 `map`，应该是给定 `Check[E, A, B]`，以及函数 `B => A => F[C]`，其中 `A => F[C]` 即为 `Check[E, A, C]`：

```Scala
sealed trait Check[E, A, B] {
  def apply(value: A)(implicit s: Semigroup[E]): Validated[E, B]

  def flatMap[C](f: B ⇒ Check[E, A, C]): Check[E, A, C] = FlatMap(this, f)
}

final case class FlatMap[E, A, B, C](check: Check[E, A, B], f: B ⇒ Check[E, A, C]) extends Check[E, A, C] {
  def apply(value: A)(implicit s: Semigroup[E]): Validated[E, C] =
    check(value).withEither(_.flatMap(b ⇒ f(b)(value).toEither))
}
```

* `flatMap` 实现与 `map` 基本相同，唯一的区别在于 `Validated` 并没有 `flatMap` 函数，因此需要将其暂时转换为 `Either` 后，再使用 `Either.flatMap`；
* `flatMap` 在这里比较奇怪，暂时没想到它的使用场景；

### `andThen`

将多个 `Check` 串联是一个很有用的功能，这与函数组合的 `andThen` 很像：

```Scala
val f: A => B = ???
val g: B => C = ???
val h: A => C = f andThen g
```

`Check` 本质是函数 `A => Validated[E, B]`，因此可以为它定义类似的 `andThen` 函数：

```Scala
sealed trait Check[E, A, B] {
  def apply(value: A)(implicit s: Semigroup[E]): Validated[E, B]

  def andThen[C](that: Check[E, B, C]): Check[E, A, C] = AndThen(this, that)
}

final case class AndThen[E, A, B, C](first: Check[E, A, B], next: Check[E, B, C]) extends Check[E, A, C] {
  def apply(value: A)(implicit s: Semigroup[E]): Validated[E, C] =
    first(value).withEither(_.flatMap(b ⇒ next(b).toEither))
}
```

## 10.4.3 Recap

目前两个 ADT 基本已经完成实现，对代码稍加整理。

将 `Predicate` 的 `case class` 定义移动到 `Predicate` 伴生对象中，并添加一个函数，用于方便的从函数 `A => Validate[E, A]` 创建 `Predicate`：

```Scala
import cats.Semigroup
import cats.data.Validated
import cats.data.Validated.{Invalid, Valid}
import cats.syntax.validated._
import cats.syntax.semigroup._
import cats.syntax.apply._

sealed trait Predicate[E, A] {
  import Predicate._

  def and(that: Predicate[E, A]): Predicate[E, A] = And(this, that)

  def or(that: Predicate[E, A]): Predicate[E, A] = Or(this, that)

  def apply(value: A)(implicit s: Semigroup[E]): Validated[E, A] =
    this match {
      case Pure(f)          ⇒ f(value)
      case And(left, right) ⇒ (left(value), right(value)).mapN((_, _) ⇒ value)
      case Or(left, right)  ⇒
        left(value) match {
          case Valid(v)   ⇒ v.valid
          case Invalid(x) ⇒
            right(value) match {
              case Valid(v)   ⇒ v.valid
              case Invalid(y) ⇒ (x |+| y).invalid
            }
        }
    }
}

object Predicate {
  final case class And[E, A](left: Predicate[E, A], right: Predicate[E, A]) extends Predicate[E, A]

  final case class Or[E, A](left: Predicate[E, A], right: Predicate[E, A]) extends Predicate[E, A]

  final case class Pure[E, A](f: A ⇒ Validated[E, A]) extends Predicate[E, A]

  def apply[E, A](f: A ⇒ Validated[E, A]): Predicate[E, A] =
    Pure(f)

  def lift[E, A](error: E, f: A ⇒ Boolean): Predicate[E, A] =
    Pure { a ⇒
      if (f(a)) a.valid
      else error.invalid
    }

}
```

用同样方式整理 `Check`：

```Scala
package com.satansk.cats.validation

import cats.Semigroup
import cats.data.Validated

/**
  * Author: Kyle Song
  * Date:   下午2:18 at 18/3/31
  * Email:  satansk@hotmail.com
  */
sealed trait Check[E, A, B] {
  import Check._

  def apply(value: A)(implicit s: Semigroup[E]): Validated[E, B]

  def map[C](f: B ⇒ C): Check[E, A, C] = Map(this, f)

  def flatMap[C](f: B ⇒ Check[E, A, C]): Check[E, A, C] = FlatMap(this, f)

  def andThen[C](that: Check[E, B, C]): Check[E, A, C] = AndThen(this, that)
}

object Check {
  def apply[E, A](p: Predicate[E, A]): Check[E, A, A] = PurePredicate(p)

  def apply[E, A](f: A ⇒ Validated[E, A]): Check[E, A, A] = Pure(f)

  final case class AndThen[E, A, B, C](first: Check[E, A, B], next: Check[E, B, C]) extends Check[E, A, C] {
    def apply(value: A)(implicit s: Semigroup[E]): Validated[E, C] =
      first(value).withEither(_.flatMap(b ⇒ next(b).toEither))
  }

  final case class Map[E, A, B, C](check: Check[E, A, B], f: B ⇒ C) extends Check[E, A, C] {
    def apply(value: A)(implicit s: Semigroup[E]): Validated[E, C] =
      check(value).map(f)
  }

  final case class FlatMap[E, A, B, C](check: Check[E, A, B], f: B ⇒ Check[E, A, C]) extends Check[E, A, C] {
    def apply(value: A)(implicit s: Semigroup[E]): Validated[E, C] =
      check(value).withEither(_.flatMap(b ⇒ f(b)(value).toEither))
  }

  final case class PurePredicate[E, A](p: Predicate[E, A]) extends Check[E, A, A] {
    def apply(value: A)(implicit s: Semigroup[E]): Validated[E, A] = p(value)
  }

  final case class Pure[E, A](f: A ⇒ Validated[E, A]) extends Check[E, A, A] {
    def apply(value: A)(implicit s: Semigroup[E]): Validated[E, A] = f(value)
  }

}
```

目前为止，已经基本实现了最初的设计目标，但还有如下问题：

1. `Predicate` 可以抽象为 `Monoid`（`and` `or` 即为 `combine`），而 `Check` 可以抽象为 `Monad`；
2. `Check` 实现好像什么都没做，实际工作基本是 `Predicate` 和 `Validate` 做的；

### 实战

暂时先不优化，先用 `Check` 和 `Predicate` 实现开头例子中的校验任务：

1. 用户名至少 4 个字符，且只能是字母或数字；
2. 电子邮件必须包含 `@` 符号，且 `@` 左侧不能为空字符串，右侧至少 3 个字符，且必须包含 `.`；

下面是可能用到的辅助函数：

```
import cats.data.NonEmptyList

type Errors = NonEmptyList[String]

def error(s: String): Errors = NonEmptyList(s, Nil)

def longerThan(n: Int): Predicate[Errors, String] =
  Predicate.lift(
    error(s"Must be longer than $n characters"),
    str ⇒ str.size > n
  )

val alphanumeric: Predicate[Errors, String] =
  Predicate.lift(
    error(s"Must be all alphanumeric characters"),
    _.forall(_.isLetterOrDigit)
  )

def contains(c: Char): Predicate[Errors, String] =
  Predicate.lift(
    error(s"Must contain the character $c"),
    _.contains(c)
  )

def containsOnce(c: Char): Predicate[Errors, String] =
  Predicate.lift(
    error(s"Must contain the character $c only once"),
    _.count(_ == c) == 1
  )
```

#### 名字校验

名字校验比较简单，直接用 `and` 组合两个 `Predicate` 即可：

```Scala
import Check._

val checkUserName: Check[Errors, String, String] =
  PurePredicate(longerThan(4) and alphanumeric)
```

使用如下：

```Scala
// Invalid(NonEmptyList(Must be longer than 3 characters))
checkUserName("Kun")

// Valid(SongKun)
checkUserName("SongKun")

// Valid(Kun123)
checkUserName("Kun123")

// Invalid(NonEmptyList(Must be all alphanumeric characters))
checkUserName("Kun.123")
```

#### 邮件校验




#### `User` 校验


