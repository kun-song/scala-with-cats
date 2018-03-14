# Chapter 5 Monad Transformers

Monad 就像巧克力，一旦尝了一口，就让人欲罢不能。万事有正反，巧克力会让人长胖，而 Monad 则可能会通过 **嵌套的 for 解析** 使代码膨胀。

假设现在要从数据库中查询用户信息，

1. 因为用户可能存在，也可能不存在，因此需要返回一个 `Option[User]`
2. 由于与数据库的通信可能会由于多种原因失败（网络错误、认证失败等），因此需要用 `Either` 包裹下 `Option`

最后的返回类型为 `Either[Error, Option[User]]`，现在问题来了，要获取 `User`，需要嵌套使用 `flatMap`，或者嵌套使用 for 解析：

```Scala
def lookupUserName(id: Long): Either[Error, Option[String]] =
  for {
    optUser <- lookupUser(id)
  } yield {
    for { user <- optUser } yield user.name
  }
```

嵌套两层还行，当 `Monad` 嵌套多了以后，不管是嵌套的 `flatMap` 还是嵌套的 for 解析，都会变得非常难以理解。