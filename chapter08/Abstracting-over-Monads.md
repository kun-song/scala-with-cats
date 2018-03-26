# 8.2 Abstracting over Monads

改造完 `UptimeClient` 后，还需要改造 `UptimeService`，否则无法编译：

```Scala
import scala.language.higherKinds

import cats.Applicative
import cats.instances.list._  // for Traverse[List]
import cats.syntax.traverse._ 
import cats.syntax.functor._  // for map

class UptimeService[F[_]: Applicative](client: UptimeClient[F]) {
  def getTotalUptime(hostnames: List[String]): F[Int] =
    hostnames.traverse(client.getUptime).map(_.sum)
}
```

* 使用 `Traverse.traverse` 的前提是 `F` 必须有 `Applicative` 实例，使用 context bound 施加该约束；

现在前面的单元测试无需任何修改就可以运行了：

```Scala
def testTotalUptime() = {
  val hosts = Map("host1" → 1, "host2" → 2)
  val client = new TestUptimeClient2(hosts)
  val service = new UptimeService(client)

  val actual = service.getTotalUptime(hosts.keys.toList)
  val expect = hosts.values.sum

  assert(actual == expect)
}

testTotalUptime()
```
