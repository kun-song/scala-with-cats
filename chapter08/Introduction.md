# Chapter 8 Case Study: Testing Asynchronous Code

第一个案例比较简单：通过将异步函数同步化，来简化异步函数的单元测试。

先扩展下第 7 章中服务器运行时间查询的例子，现在该功能有两个模块组成，首先是负责查询的客户端 `UptimeClient`：

```Scala
import scala.concurrent.Future

trait UptimeClient {
  def getUptime(host: String): Future[Int]
}
```

以及负责响应的服务端 `UptimeService`：

```Scala
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

import cats.instances.list._    // for Traverse[List]
import cats.instances.future._  // for Applicative[Future]
import cats.syntax.traverse._

class UptimeService(client: UptimeClient) {
  def getTotalUptime(hostnames: List[String]): Future[Int] =
    hostnames.traverse(client.getUptime).map(_.sum)
}
```

通过将 `UptimeClient` 设为 `trait`，可以在单元测试中队 `UptimeClient` 打桩，例如可以直接返回模拟数据，而非实际请求服务器：

```Scala
class TestUptimeClient(hosts: Map[String, Int]) extends UptimeClient {
  def getUptime(host: String): Future[Int] =
    Future.successful(hosts.getOrElse(host, 0))
}
```

若要测试 `UptimeService` 的求和功能，这与 `UptimeClient` 具体实现没有关系，因此可以用 `TestUptimeClient`：

```Scala
def testTotalUptime = {
  val hosts = Map("host1" → 1, "host2" → 2)
  val client = new TestUptimeClient(hosts)
  val service = new UptimeService(client)

  val actual = service.getTotalUptime(hosts.keys.toList)
  val expect = hosts.values.sum

  assert(actual == expect)
}
```

但实际运行会报错，因为 `getTotalUptime` 返回 `Future[Int]`，而 `expect` 是 `Int`，两者类型不匹配。

当然可以用 `Await` 获取 `getTotalUptime` 返回值中的 `Int`，但这里我们通过将 `UptimeClient.getUptime` 修改为同步代码解决。
