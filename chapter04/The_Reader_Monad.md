# 4.8 The Reader Monad

`Reader` monad 用于序列化 **依赖某相同输入** 的一系列函数，`Reader` 实例包裹的函数 **仅接受一个入参**。

`Reader` 的典型易用场景为 **依赖注入**，假如有一系列函数都 **依赖** 某外部配置，则可利用 `Reader` 将这些函数组合为一个函数，该函数接受 **共同依赖的配置** 作为输入，并按顺序运行组合的函数。

## 4.8.1 Creating and Unpacking Readers

通过 `Reader.apply` 函数创建 `Reader[A, B]` 实例，该函数接受 `A => B` 函数作为参数：

```Scala
import cats.data.Reader

case class Cat(name: String, favoriteFood: String)

val catName: Reader[Cat, String] = Reader(cat ⇒ cat.name)
```

通过 `Reader.run` 提取封装在 `Reader` 中的 `A => B` 函数：

```Scala
catName.run(Cat("Mike", "mantou"))
// or
catName(Cat("Mike", "mantou"))
```

## 4.8.2 Composing Readers

`Reader` 真正强大之处在于 `map` 和 `flatMap` 函数，它们代表了不同的 **函数组合** 方式。

使用 `Reader` 的一般套路如下：

1. 创建一系列接受 **相同输入** 的函数；
2. 使用 `map` 和 `flatMap` 进行函数组合，形成一个函数；
3. 调用 `Reader.run`，传入 **相同输入**，运行所有函数；

`map` 接受 `Reader[A, B]` 中 `A => B` 函数的 **计算结果**，从而扩展 `Reader`：

```Scala
val greet: Reader[Cat, String] = catName.map(name ⇒ s"hello $name")

greet(Cat("Mike", "mantou"))
```

`flatMap` 用于组合接受 **相同输入** 的另一个 `Reader`：

```Scala
import cats.data.Reader

case class Cat(name: String, favoriteFood: String)

val catName: Reader[Cat, String] = Reader(cat ⇒ cat.name)

// map
val greet: Reader[Cat, String] = catName.map(name ⇒ s"hello $name")

val feed: Reader[Cat, String] = Reader(cat ⇒ s"feed you a ${cat.favoriteFood}")

// flatMap + map
val greetAndFeed: Reader[Cat, String] =
  for {
    g ← greet
    f ← feed
  } yield s"$g. $f"

// hello Mike. feed you a mantou
greetAndFeed(Cat("Mike", "mantou"))
```

## 4.8.3 Exercise: Hacking on Readers

前面说过 `Reader` 可用于组合接受 **相同输入** 的多个函数，假设要构建一个网站登录系统，相同输入是：

```Scala
case class Db(
  usernames: Map[Int, String],
  passwords: Map[String, String]
)
```

因为后面所有函数的输入都是 `Db`，因此为 `Reader[Db, A]` 创建一个别名，简化代码：

```Scala
type DbReader[A] = Reader[Db, A]
```

现在实现两个函数：`findUsername` 根据用户 Id 查找用户名字，`checkPassword` 则检测给定的用户名/密码是否正确：

```Scala
//def findUsername(userId: Int): DbReader[Option[String]] =
//  Reader(_.usernames.find(_._1 == userId).map(_._2))
//
//def checkPassword(username: String, password: String): DbReader[Boolean] =
//  Reader(_.passwords.find(_._1 == username).map(_._2).exists(_ == password))

def findUsername(userId: Int): DbReader[Option[String]] =
  Reader(_.usernames.get(userId))

def checkPassword(username: String, password: String): DbReader[Boolean] =
  Reader(_.passwords.get(username).contains(password))
```

最后组合以上两个函数为 `checkLogin`，检查给定的用户 Id 和密码是否正确：

```Scala
import cats.data.Reader
import cats.syntax.applicative._

def checkLogin(userId: Int, password: String): DbReader[Boolean] =
  for {
    nameOption ← findUsername(userId)
    valid      ← nameOption.map(name ⇒ checkPassword(name, password)).getOrElse(false.pure[DbReader])
  } yield valid
```

最后，`checkLogin` 使用方式如下：

```Scala
val db = Db(
  Map(1 → "a", 2 → "b"),
  Map("a" → "123", "b" → "234"))

checkLogin(1, "123")(db)
```

4.8.4 When to Use Readers?

`Reader` 可以做 **依赖注入**，将应用分解为 `Reader` 实例，然后用 `map` `flatMap` 将它们组合为一个 `Reader` 实例，最后注入依赖，运行整个应用。

但 Scala 中有很多实现依赖注入的方式，from simple techniques like methods with multiple parameter lists, through implicit parameters and type classes, to complex techniques like the cake pattern and DI frameworks.

适合用 `Reader` 做依赖注入的场景有：

* we are constructing a batch program that can easily be represented by a function;
* we need to defer injection of a known parameter or set of parameters;
* we want to be able to test parts of the program in isolation.

当场景更加复杂后，依赖非常多，或者应用本身无法表示为 **纯函数** 时，其他依赖注入方式更加适合。

>Kleisli Arrows
>
>Cats 中，`Reader` 使用更加通过的 `Kleisli` 实现。

