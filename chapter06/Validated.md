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

有了 `Semigroupal` instance 后，可以使用所有 `Semigroupal` 本身具有的函数 or apply syntax 对它们进行组合。

组合的前提：当前作用域必须有合适的 `Semigroupal` 实例！

回忆下 `Semigroupal` 定义：

```Scala
@typeclass trait Semigroupal[F[_]] {
  def product[A, B](fa: F[A], fb: F[B]): F[(A, B)]
}
```

只能使用 single parameter type constructor 构建 `Semigroupal` 实例，而 `Validated` 有两个类型参数，只能使用 type alias：

```Scala
import cats.data.Validated

type AllErrorsOr[A] = Validated[String, A]
```
* `AllErrorsOr` 只有一个类型参数，可以用于创建 `Semigroupal[AllErrorsOr]` 实例

`Validated` 内部使用 `Semigroup` 收集错误，若作用域中无合适的 `Semigroup` 实例，会报错：

```Scala
Semigroupal[AllErrorsOr]

Error:(6, 79) could not find implicit value for parameter instance: cats.Semigroupal[A$A137.this.AllErrorsOr]
def get$$instance$$res0 = /* ###worksheet### generated $$end$$ */ Semigroupal[AllErrorsOr]
                                                                             ^
```

因为 `AllErrorsOr` 错误类型为 `String`，因此导入 `Semigroup[String]` 实例，即可消除报错：

```Scala
import cats.instances.string._  // for Semigroup
```

作用域中所有需要的 `implicit` 实例后，即可收集错误，可以用 `Semigroupal` 伴生对象中的方法，也可以用 `cats.syntax.apply` 定义的语法，这里用 apply syntax，因为它更灵活一些：

```Scala
import cats.data.Validated
import cats.instances.string._
import cats.syntax.validated._
import cats.syntax.apply._

type AllErrorsOr[A] = Validated[String, A]

// cats.data.Validated[String,(Int, Int)] = Invalid(Error 1Error 2)
(
  "Error 1".invalid[Int],
  "Error 2".invalid[Int]
).tupled
```

* `invalid` 来自 `cats.syntax.validated`
* `tupled` 来自 `cats.syntax.apply`

`tupled` 结果为 `Invalid(Error 1Error 2)`，`Error 1` 和 `Error 2` 糊在一起了，因此不太适合用 `String`，一般用 `List` 或 `Vector` 用作 `Validated` 的第一个类型参数：

```Scala
import cats.data.Validated
import cats.instances.list._    // for Semigroupal
import cats.syntax.validated._  // for invalid
import cats.syntax.apply._      // for tupled

type AllErrorsOr[A] = Validated[List[String], A]

// cats.data.Validated[List[String],(Int, Int)] = Invalid(List(Error 1, Error 2))
(
  List("Error 1").invalid[Int],
  List("Error 2").invalid[Int]
).tupled
```

`tupled` 结果为 `List(Error 1, Error 2)`，比 `Error 1Error 2` 清晰多了。

The `cats.data` package also provides the `NonEmptyList` and `NonEmptyVector` types that prevent us failing without at least one error:

```Scala
import cats.data.{NonEmptyList, Validated}
import cats.syntax.validated._
import cats.syntax.apply._

type AllErrorsOr[A] = Validated[List[String], A]

// cats.data.Validated[List[String],(Int, Int)] = Invalid(NonEmptyList(Error 1, Error 2))
(
  NonEmptyList.of("Error 1").invalid[Int],
  NonEmptyList.of("Error 2").invalid[Int]
).tupled
```

## 6.4.3 Methods of Validated

### `map` `leftMap` `bimap`

`Validated` 提供了与 `Either`、`cats.syntax.either` 非常相似的方法：

```Scala
import cats.syntax.validated._

// Validated[Nothing,Int] = Valid(666)
val a = 111.valid.map(_ * 6)

// Validated[String,Nothing] = Invalid(xxx)
val b = "x".invalid.leftMap(_ * 3)

// Validated[String,Int] = Valid(1110)
val c = 111.valid[String].bimap(_ * 3, _ * 10)
// Validated[String,Int] = Invalid(xxx)
val d = "x".invalid[Int].bimap(_ * 3, _ * 10)
```

* `map` 转换 `valid`
* `leftMap` 转换 `invalid`
* `bimap` 转换 `valid` + `invalid`

### `toEither`

因为 `Validated` 是 `Semigroupal` 而非 `Monad`，因此没有 `flatMap`，但可以通过 `toEither` 和 `toValidated` 两个函数在 `Validated` 和 `Either` 之间任意转换（`toValidated` 来自 `cats.syntax.either`）：

```Scala
import cats.syntax.validated._
import cats.syntax.either._

// Validated[String,Int] = Valid(111)
val a = 111.valid[String]

// Either[String,Int] = Right(111)
val b = 111.valid[String].toEither

// Validated[String,Int] = Valid(111)
val c = 111.valid[String].toEither.toValidated
```
* `toEither.toValidated` 相当于啥都没做

### `withEither` `withValidated`

`withEither` 可以将 `Validated` 暂时转换为 `Either`，并且执行完后，立即转换回 `Validated`：

```Scala
import cats.syntax.validated._

// Validated[String,Int] = Valid(666)
val a = 111.valid[String].withEither(_.map(_ * 6))

// Validated[String,Int] = Valid(666)
val b = 111.valid[String].withEither(_.flatMap(n ⇒ Right(n * 6)))
```

`cats.syntax.either` 为 `Either` 定义了类型的 `withValidated` 函数。

### `ensure`

```Scala
import cats.syntax.validated._

// Validated[String,Int] = Valid(111)
val a = 111.valid[String].ensure("can't be negative!")(_ > 0)

// Validated[String,Int] = Invalid(can't be negative!)
val b = -11.valid[String].ensure("can't be negative!")(_ > 0)
```

### `getOrElse` `fold`

```Scala
"fail".invalid[Int].getOrElse(0)
// res26: Int = 0

"fail".invalid[Int].fold(_ + "!!!", _.toString)
// res27: String = fail!!!
```

## 6.4.4 Exercise: Form Validation

**题目**：

使用 `Validated` 实现表单验证，用户输入信息类型为 `Map[String, String]`，我们从中获取用户的 `name` 和 `age`，并创建 `User` 对象：

```Scala
case class User(name: String, age: Int)
```

`name` 和 `age` 必须满足：

* the name and age must be specified;
* the name must not be blank;
* the age must be a valid non-negative integer.

若所有检验通过，则返回 `User` 对象，否则返回一个 `List`，包含 **所有** 错误信息。

**思路**：要实现该功能，需要结合 `Either` 和 `Validated`

首先定义用到的类型别名：

```Scala
import cats.data.Validated

type FormData = Map[String, String]

type FailFast[A] = Either[List[String], A]
type FailSlow[A] = Validated[List[String], A]
```

### 1. `getValue`

实现 `getValue`，接受 `Map[String, String]` 和一个 `name`，从 `Map` 中读取 `name` 对应的值：

```Scala
def getValue(name: String)(data: FormData): FailFast[String] =
  data.get(name).toRight(List(s"$name field not specified!"))
```

`getValue` 使用如下：

```Scala
val getName = getValue("name") _

// FailFast[String] = Right(Mike)
getName(Map("name" → "Mike"))

// FailFast[String] = Left(List(name field not specified!))
getName(Map())
```

### 2. `parseInt`

`parseInt` 解析 `String`，获取 `Int`：

```Scala
import cats.syntax.either._

def parseInt(name: String)(data: String): FailFast[Int] =
  Either.catchOnly[NumberFormatException](data.toInt)
    .leftMap(_ ⇒ List(s"$name must be an integer!"))
```
* 使用 `Either.catchOnly` 捕获 `toInt` 的异常，并用 `leftMap` 将左侧的 `NumerFormatException` 转换为 `List[String]`

`parseInt` 使用如下：

```Scala
// FailFast[Int] = Right(11)
parseInt("age")("11")

// FailFast[Int] = Left(List(age must be an integer!))
parseInt("age")("xx")
```

### 3. `nonBlank` `nonNegative`

`nonBlank` 检验字符串，`nonNegative` 校验数字：

```Scala
def nonBlank(name: String)(data: String): FailFast[String] =
  Right(data).ensure(List(s"$name can't be blank!"))(_.nonEmpty)

def nonNegative(name: String)(data: Int): FailFast[Int] =
  Right(data).ensure(List(s"$name can't be negative!"))(_ > 0)
```

使用如下：

```Scala
// FailFast[String] = Right(Mike)
nonBlank("name")("Mike")
// FailFast[String] = Left(List(name can't be blank!))
nonBlank("name")("")

// FailFast[Int] = Right(11)
nonNegative("age")(11)
// FailFast[Int] = Left(List(age can't be negative!))
nonNegative("age")(-1)
```

### 4. `readName` `readAge`

基于 `getValue` `parseInt` `nonBlank` `nonNegative` 实现 `readName` 和 `readAge`：

```Scala
def readName(data: FormData): FailFast[String] =
  getValue("name")(data)
    .flatMap(nonBlank("name"))

def readAge(data: FormData): FailFast[Int] =
  getValue("age")(data)
    .flatMap(nonBlank("age"))
    .flatMap(parseInt("age"))
    .flatMap(nonNegative("age"))
```

* `readName` 必须满足 `nonBlank` 一条规则
* `readAge` 必须满足 `nonBlank` 和 `nonNegative` 两条规则
* 使用 `flatMap` 串联规则，因为实际中可能有非常非常多的规则，但只要有一条规则不满足，就立即返回，这符合业务要求

使用如下：

```Scala
readName(Map("name" -> "Dade Murphy"))
// res41: FailFast[String] = Right(Dade Murphy)

readName(Map("name" -> ""))
// res42: FailFast[String] = Left(List(name cannot be blank))

readName(Map())
// res43: FailFast[String] = Left(List(name field not specified))

readAge(Map("age" -> "11"))
// res44: FailFast[Int] = Right(11)

readAge(Map("age" -> "-1"))
// res45: FailFast[Int] = Left(List(age must be non-negative))

readAge(Map())
// res46: FailFast[Int] = Left(List(age field not specified))
```

### 5. 组合 `readName` 和 `readAge`

使用 `Semigroupal` 组合 `readName` 和 `readAge`，产生一个 `User` 对象：

```Scala
def readUser(data: FormData): FailSlow[User] =
  (
    readName(data).toValidated,
    readAge(data).toValidated
  ).mapN(User.apply)
```

>在 `Validated` 和 `Either` 之间转来转去很麻烦，一般是这样的，对某个场景而言，fail-fast or accumulating error-handling 两者，只有一种比较合适，所以根据场景选择合适的错误处理策略，在场景之间交互时，再进行转换即可。
