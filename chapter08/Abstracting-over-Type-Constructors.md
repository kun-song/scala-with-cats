# 8.1 Abstracting over Type Constructors

继承 `UptimeClient` 特质，分别为生产环境、测试环境创建两个子特质，`getUptime` 函数分别为 **异步** 和 **同步**：

```Scala
trait RealUptimeClient extends UptimeClient {
  override def getUptime(host: String): Future[Int]
}

trait TestUptimeClient extends UptimeClient {
  override def getUptime(host: String): Int
}
```
* 两个函数返回值分别是 `Future[Int]` 和 `Int`

问题是 `UptimeClient.getUptime` 返回类型应该是什么呢？它必须兼容 `Future[Int]` 和 `Int`，但这两者没有公共父类，可以用 `Id` 解决：

```Scala
trait UptimeClient[F[_]] {
  def getUptime(host: String): F[Int]
}

trait RealUptimeClient extends UptimeClient[Future] {
  override def getUptime(host: String): Future[Int]
}

trait TestUptimeClient extends UptimeClient[Id] {
  override def getUptime(host: String): Id[Int]
}
```

因为 `Id[Int]` 就是 `Int` 的别名，因此 `TestUptimeClient` 返回类型可以直接写 `Int`：

```Scala
trait TestUptimeClient extends UptimeClient[Id] {
  override def getUptime(host: String): Int
}
```

重新实现打桩 `UptimeClient` 如下：

```Scala
class TestUptimeClient2(hosts: Map[String, Int]) extends UptimeClient[Id] {
  def getUptime(host: String): Int = hosts.getOrElse(host, 0)
}
```
