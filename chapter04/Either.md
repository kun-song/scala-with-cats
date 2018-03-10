# 4.4 Either

本节介绍 Scala 标准库中的一个 monad，即 `Either`，在 Scala 2.11 以及更老的版本中，大家并不认为 `Either` 是 monad，因为它没有 `map` 和 `flatMap` 函数，但在 Scala 2.12 后，`Either` 彻底变成了 monad。

4.4.1 Left and Right Bias

Scala 2.11 中，`Either` 没有 `map` 和 `flatMap` 函数，使其在 for 解析中非常难用，在每个 generator 后面都要加个 `.right` 后缀：

```Scala
val e1: Either[String, Int] = Right(10)
val e2: Either[String, Int] = Right(32)

for {
  a <- e1.right
  b <- e2.right
} yield a + b
```

Scala 2.12 重新设计了 `Either`，现在认为 right side 表示成功，因此支持直接在 `Either` 上执行 `map` 和 `flatMap`：

```Scala
val e1: Either[String, Int] = Right(10)
val e2: Either[String, Int] = Right(32)

for {
  a <- e1
  b <- e2
} yield a + b
```

Cats 通过 `cats.syntax.either` 为老版本 Scala 提供了 right-biased 的 `Either`：

```Scala
import cats.syntax.either._ // for map and flatMap

for {
  a <- e1
  b <- e2
} yield a + b
```

注意，在新版 Scala 中使用 `import cats.syntax.either._` 不会影响标准库中 `Either` 的正常使用。

## 4.4.2 Creating Instances

可以直接用 `Left.apply` 和 `Right.apply` 创建 `Either` 实例：

```Scala
// Right[Nothing,Int]
val a = Right(1)

// Left[Int,Nothing]
val b = Left(1)
```

* `Either` 需要两个 type parameter，若未明确指定，则 Scala 编译器将其推断为 `Nothing`，例如上面例子中的 `Right(1)`，Scala 编译器可以推断出右侧类型为 `Int`，但由于未明确指定左侧类型，所以最终 `a` 的类型为 `Right(Nothing, Int)`，同理 `b` 类型为 `Left(Int, Nothing)`。
* 使用 `Right.apply` 创建的值，若未明确指定类型，则为 `Right(A, B)`；
* 使用 `Left.apply` 创建的值，若未明确指定类型，则为 `Left(A, B)`；

若想要返回 `Either(A, B)`，而非 `Right(A, B)` 和 `Left(A, B)`，可以通过 `cats.syntax.either._` 中定义的 smart constructor 创建：

```Scala
import cats.syntax.either._

// Either[String, Int]
val a = 1.asRight[String]

// Either[String, Int]
val b = "wrong".asLeft[Int]
```

返回值是 `Either` 而非更具体的 `Right` `Left` 可以避免一些类型推断错误，例如：

```Scala
import cats.syntax.either._

def countPositive(xs: List[Int]): Either[String, Int] =
  xs.foldLeft(Right(0)) { (e, i) ⇒
   if (i > 0) e.map(_ + 1)
   else Left("Negative")
  }
```

上面代码将编译失败，原因为：

* 因为 `foldLeft` 第一个参数类型为 `Right`，所以编译器推断出 `e` 的类型为 `Right`；
* 因为 `Right.apply` 未指定左侧类型，编译器将其推断为 `Nothing`；

所以最后 `e` 类型被推断为 `Right[Nothing, Int]`， 所以 `e.map(_ + 1)` 这句编译器将在 `implicit` 作用域中查找 `Monad[Right[Nothing, Int]]` 实例，然而找不到，所以编译失败。

使用 `asRight` 智能构造器可以避免上述问题：

```Scala
import cats.syntax.either._

def countPositive(xs: List[Int]): Either[String, Int] =
  xs.foldLeft(0.asRight[String]) { (e, i) ⇒
   if (i > 0) e.map(_ + 1)
   else Left("Negative")
  }
```

* `asRight` 用户只需指定 **另外一侧** 的类型，不需要指定两边类型，更方便；
* `asRight` 返回类型为 `Either` 而非 `Right`；

### `cats.syntax.either` 中的扩展方法

`cats.syntanx.either` 中定义了一些标准库 `Either` 伴生对象中没有扩展方法，使用非常方便。

#### 捕获错误（从表达式创建 `Either`）

`catchOnly` `catchNotFatal` 用于捕获表达式可能出现的异常，返回结果类型为 `Either`：

```Scala
// Either[NumberFormatException,Int]
Either.catchOnly[NumberFormatException]("aa".toInt)

// Either[NumberFormatException,Int]
Either.catchOnly[NumberFormatException]("123".toInt)

// Either[ArithmeticException,Int]
Either.catchOnly[ArithmeticException](10 / 0)

// Either[Throwable,Nothing]
Either.catchNonFatal(sys.error("error"))

// Either[Throwable,Int]
Either.catchNonFatal("aa".toInt)

// Either[Throwable,Int]
Either.catchNonFatal(10 / 0)
```

#### 从其他类型创建 `Either`

```Scala

Either.fromTry(Try(10 / 0))
Either.fromOption(Some(10), "xxx")  // Right(10)
Either.fromOption(None, "a none")  // Left(a none)
```

`Either.fromOption` 第一个参数为 `Option`，第二个参数为若 `Option` 为 `None`，left side 的值。

## 4.4.3 Transforming Eithers

`cats.syntax.either` 也对 `Either` 实例进行了增强。

使用 `orElse` 和 `getOrElse` 从 `Either` 右侧提取值，或者返回一个默认值：

```Scala
"Wrong!".asLeft[Int].orElse(20.asRight[Int])  // Right(20)
"Wrong!".asLeft[Int].getOrElse("11".toInt)  // 11
111.asRight[String].getOrElse(-1)  // 111
```

`ensure` 可以对 `Either` 右侧施加断言：

```Scala
-1.asRight[String].ensure("Must be positive!")(_ > 0)  // Left(Must be positive!)

1.asRight[String].ensure("Must be positive!")(_ > 0)  // Right(1)
```

`recover` 和 `recoverWith` 为 `Either` 提供类似 `Future` 的 error handling 机制：

```Scala
// Right(-1)
"error".asLeft[Int].recover {
  case _ ⇒ -1
}

// Right(-1)
"error".asLeft[Int].recoverWith {
  case _ ⇒ Right(-1)
}
```

除 `map` 外，还提供 `leftMap` `bimap` 作为补充：

```Scala
"wrong".asLeft[Int].leftMap(_ + "!!!")  // Left("wrong!!!")

"wrong".asLeft[Int].bimap(_.reverse, _ * 6)  // Left(gnorw)
111.asRight[String].bimap(_.reverse, _ * 6)  // Right(666)
```

使用 `swap` 可以交换左右两侧的值：

```Scala
"error".asLeft[Int]  // Either[String,Int] = Left(error)

"error".asLeft[Int].swap  // Either[Int,String] = Right(error)
```

## 4.4.4 Error Handling

`Either` 一般用于实现 **fail-fast error handling**，通过使用 `flatMap` 序列化计算，只要中间某步失败，则停止整个计算：

```Scala
// Either[String,Int] = Left(divide 0)
for {
  x ← 111.asRight[String]
  y ← 0.asRight[String]
  c ← if (y == 0) "divide 0".asLeft[Int]
      else (x / y).asRight[String]
} yield c * 100
```

当 `Either` 用于 error handling 时，需要决定用什么类型表示错误，最简单的方式是用 `Throwable` 表示：

```Scala
type Result[A] = Either[Throwable, A]
```

这样获得的语义类似 `Try`，但由于 `Throwable` 是一个非常广泛的类型，因此很难知道到底发生了什么错误。

另一种方式是将应用中可能出现的错误用 algebraic data type 建模，假设应用中用户登陆可能出现 3 种不同错误：

```Scala
sealed trait LoginError extends Product with Serializable

final case class UserNotFound(username: String) extends LoginError
final case class PasswordIncorrect(username: String) extends LoginError
case object UnexpectedError extends LoginError
```

则可以用 `Either` 将错误表示为：

```Scala
case class User(username: String, password: String)

type LoginResult = Either[LoginError, User]
```

这就避免了使用 `Throwable` 表示错误的局限性，可以通过模式匹配穷举可能出现的错误，并给出应对方案：

```Scala
def handleError(error: LoginError): Unit =
  error match {
    case UserNotFound(name) ⇒ println(s"user not found: $name")
    case PasswordIncorrect(name)  ⇒ println(s"password incorrect: $name")
    case UnexpectedError  ⇒ println("we know nothing!")
  }
```

至此，错误处理方式如下：

```Scala
val r1: LoginResult = User("songkun", "123").asRight
val r2: LoginResult = PasswordIncorrect("xx").asLeft

r1.fold(handleError, println)
r2.fold(handleError, println)
```
