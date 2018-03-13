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

前面说过 `Reader` 可用于组合接受 **相同输入** 的多个函数，假设现在的相同输入是：

```Scala
case class Db(
  usernames: Map[Int, String],
  passwords: Map[String, String]
)
```

