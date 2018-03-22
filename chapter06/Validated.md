# 6.4 Validated

前面展示了以 `Either` 实现 fail-fast error-handling，因为 `Either` 是 `Monad`，因此其 `product` 与 `flatMap` 语义完全相同。

更进一步，由于 Monad 的 `product` 是用 `flatMap` 实现的，因此使用 `Monad` 根本无法实现 accumulating error-handling！

>注意，因为 `Monad` 继承了 `Semigroupal`，因此对于类型 `A`，只要实现 `Monad[A]` 实例即可，不需要单独实现 `Semigroupal[A]` 实例。

幸运的是 Cats 提供了 `Validated` 类型，且对该类型，只有 `Semigroupal[Validated]` 实例，而无 `Monad[Validated]` 实例，因此 `Semigroupal[Validated]` 可以自由地实现 accumulating 的 `product` 函数：

```Scala
import cats.Semigroupal
import cats.data.Validated
import cats.instances.list._


type AllErrorsOr[A] = Validated[List[String], A]

// AllErrorsOr[(Nothing, Nothing)] = Invalid(List(error 1, error 2))
Semigroupal[AllErrorsOr].product(
  Validated.invalid(List("error 1")),
  Validated.invalid(List("error 2"))
)
```

`Validated` 完美补充了 `Either`，两者分别实现了两种常见的错误处理策略：

* `Validated`: accumulating error-handling
* `Either`: fail-fast error-handling

## 6.4.1 Creating Instances of Validated

`Validated` 有两个子类：`Valid` 和 `Invalid`，它们分别于 `Either` 的 `Right` `Left` 有点类似。

有很多创建 `Valid` 和 `Invalid` 实例的方式：

#### 1. `Valid.apply` 和 `Invalid.apply`

最直接的方式是用 `Valid` 和 `Invalid` 的 `apply` 方法：

```Scala
import cats.data.Validated

// a: cats.data.Validated.Valid[Int] = Valid(666)
val a = Validated.Valid(666)

// cats.data.Validated.Invalid[List[String]] = Invalid(List(error))
val b = Validated.Invalid(List("error"))
```

* `a` 类型为 `Validated.Valid`
* `b` 类型为 `Validated.Invalid`

#### 2. smart constructor: `valid` 和 `invalid`

`apply` 返回的类型是 `Valid` 或 `Invalid`，而不是 `Validated`，不太方便，使用 smart constructor 可以拓宽返回类型为 `Validated`：

```Scala
import cats.data.Validated

// a: cats.data.Validated[Nothing,Int] = Valid(666)
val a = Validated.valid(666)

// b: cats.data.Validated[List[String],Nothing] = Invalid(List(error))
val b = Validated.invalid(List("error"))
```

还可以为 smart constructor 指定类型参数，从而避免返回 `Validated` 类型中的 `Nothing`：

```Scala
import cats.data.Validated

// a: cats.data.Validated[List[String],Int] = Valid(666)
val a = Validated.valid[List[String], Int](666)

// b: cats.data.Validated[List[String],Int] = Invalid(List(error))
val b = Validated.invalid[List[String], Int](List("error"))
```

#### 3. `cats.syntax.validated` 中的 `valid` 和 `invalid` 语法

```Scala
import cats.syntax.validated._

// a: cats.data.Validated[List[String],Int] = Valid(666)
val a = 666.valid[List[String]]

// b: cats.data.Validated[List[String],Int] = Invalid(List(error))
val b = List("error").invalid[Int]
```

* 返回类型也是 `Validate`，而非 `Valid` 或 `Invalid`

#### 4. `cats.syntax.applicative` 中的 `pure` 和 `cats.syntax.applicativeError` 中的 `raiseError`

```Scala
import cats.data.Validated
import cats.syntax.applicative._
import cats.syntax.applicativeError._

type ErrorOr[A] = Validated[List[String], A]

// a: cats.data.Validated[List[String],Int] = Valid(666)
val a = 666.pure[ErrorOr]

// b: cats.data.Validated[List[String],Int] = Invalid(List(error))
val b = List("error").raiseError[ErrorOr, Int]
```

#### 5. 其他类型 -> `Validated`

`Validated` 伴生对象提供了一些辅助方法，用于从 `Exception` `Either` `Option` `Try` 等创建 `Validated` 实例：

```Scala
import cats.data.Validated

import scala.util.Try

// a: cats.data.Validated[NumberFormatException,Int] = Invalid(java.lang.NumberFormatException: For input string: "a")
val a = Validated.catchOnly[NumberFormatException]("a".toInt)

// b: cats.data.Validated[Throwable,Nothing] = Invalid(java.lang.RuntimeException: Bad)
val b = Validated.catchNonFatal(sys.error("Bad"))

// c: cats.data.Validated[Throwable,Int] = Invalid(java.lang.ArithmeticException: / by zero)
val c = Validated.fromTry(Try(10 / 0))

// d: cats.data.Validated[String,Int] = Valid(666)
val d = Validated.fromEither[String, Int](Right(666))

// e: cats.data.Validated[String,Int] = Invalid(error none)
val e = Validated.fromOption[String, Int](None, "error none")
```

## 6.4.2 Combining Instances of Validated












 











