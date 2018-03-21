# 6.1 Semigroupal

`cats.Semigroupal` 用来 **组合 context**，假设有类型为 `F[A]` `F[B]` 的对象，`Semigroupal[F]` 可以将其组合为 `F[(A, B)]`，其在 Cats 中定义如下：

```Scala
trait Semigroupal[F[_]] {
  def product[A, B](fa: F[A], fb: F[B]): F[(A, B)]
}
```

* `product` 的参数 `fa` `fb` **互相独立**，可以以任意顺序计算，因此定义 `Semigroupal` instance 时更加灵活（相对 `flatMap`）

## 6.1.1 Joining Two Contexts

* `Semigroup` 用来 join values
* `Semigroupal` 用来 join contexts

例如：

```Scala
import cats.Semigroupal
import cats.instances.option._

// Option[(Int, Int)] = Some((111,6))
Semigroupal[Option].product(Some(111), Some(6))
```
* `product` 两个参数都是 `Some`，则结果为 `Option[Tuple2]`

```Scala
import cats.Semigroupal
import cats.instances.option._

// Option[(Int, Nothing)] = None
Semigroupal[Option].product(Some(111), None)
// Option[(Nothing, String)] = None
Semigroupal[Option].product(None, Some("x"))
```
* `product` 任意一个参数为 `None`，则结果为 `None`

## 6.1.2 Joining Three or More Contexts

`Semigroupal` 伴生对象中，基于 `product` 定义了能操作 **更多** 操作数的函数。

`tuple2` -> `tuple22`，类似 `prodcut`，用于组合 contexts：

```Scala
import cats.Semigroupal
import cats.instances.option._

// Option[(Int, Int, Int)] = Some((111,6,1))
Semigroupal.tuple3(Option(111), Option(6), Option(1))

// Option[(Int, String, Int)] = None
Semigroupal.tuple3(Option(111), Some("x"), Option.empty[Int])
```

`map2` -> `map22`，应用指定函数，到 2-22 个 context 中的值：

```Scala
import cats.Semigroupal
import cats.instances.option._

// Option[Int] = Some(666)
Semigroupal.map3(Option(111), Option(6), Option(1))(_ * _ * _)

// Option[String] = None
Semigroupal.map3(Option(111), Some("x"), Option.empty[Int])(_ + _ + _)
```

类似的函数还有 `contramap2` -> `contramap22`，`imap2` -> `imap22` 等。
