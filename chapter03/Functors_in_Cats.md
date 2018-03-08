# 3.5 Functors in Cats

从 type class, instances, syntax 3 个方面看 Cats 中 functor 的实现。

## 3.5.1 The `Functor` Type Class

`Functor` type class 定义在 `cats.Functor` 中，依然可以通过 `Functor.apply` 获取指定类型的 `Functor` 实例：

```Scala
import scala.language.higherKinds

import cats.Functor
import cats.instances.list._
import cats.instances.option._

// 通过 Functor.apply 获取 Functor 实例
val listFunctor: Functor[List] = Functor[List]
val optionFunctor: Functor[Option] = Functor[Option]

val xs = List(1, 2, 3)
listFunctor.map(xs)(_ + 1)  // 2, 3, 4

val o = Option(1)
optionFunctor.map(o)(_ + 1)  // Some(2)
```

`Functor` 还提供了 `lift` 函数，将 `A => B` 转换为 `F[A] => F[B]`，即将普通函数转换为在 `Functor` 上操作的函数：

```Scala
import scala.language.higherKinds

import cats.Functor
import cats.instances.list._
import cats.instances.option._

// 通过 Functor.apply 获取 Functor 实例
val listFunctor: Functor[List] = Functor[List]

val f: Int ⇒ Int = x ⇒ x * 2

// 使用 Functor.lift 将 Int => Int 转换为 List[Int] => List[Int]
val fList: List[Int] ⇒ List[Int] = listFunctor.lift(f)

val xs = List(1, 2, 3)

fList(xs)  // 2, 4, 6
```

## 3.5.2 `Functor` Syntax

`cats.syntax.functor` 提供的主要函数为 `map`，但因为 `Option` 和 `List` 原生已经定义了 `map` 函数，因此无法直接以该两种类型演示。

前面说过 `Function1` 也是 functor，而且它没有定义 `map` 函数（其 `andThen` 函数功能与 `map` 一模一样），因此可以用来演示：

```Scala
import cats.instances.function._
import cats.syntax.functor._

val f1: Int ⇒ Int = x ⇒ x + 1
val f2: Int ⇒ Int = x ⇒ x * 6

val f: Int ⇒ Int = f1.map(f2)

f(110)  // 666
```

下面例子中，我们 **不接触** 具体类型，而是全部基于 `Functor`，这样无论 *functor context*（`F` 即 context，具体可以是 `List` `Option` `Function1` 等等） 是什么，该函数都能工作：

```Scala
import scala.language.higherKinds

import cats.Functor
import cats.syntax.functor._

def doMath[F[_]](x: F[Int])(implicit functor: Functor[F]): F[Int] =
  x.map(x ⇒ x * 2)
```

* `doMath` 函数操作对象是 *context* 的 *content*，即 `List` 中的元素；

无论 `x` 是什么类型的 `Functor` 实例，`doMath` 都会将其 **内容** 乘以 2：

```Scala
import cats.instances.list._

val xs = List(1, 2, 3)

doMath(xs)  // 2, 4, 6

import cats.instances.option._

val o = Option(1)

doMath(o)  // Some(2)
```

注意 `doMath` 定义中的 `x.map(x ⇒ x * 2)`，这里利用了 `cats.syntax.functor`，为了便于理解，下面是一个简化版的 `cats.syntax.functor` 定义：

```Scala
implicit class FunctorOps[F[_], A](src: F[A]) {
  def map[B](func: A => B)(implicit functor: Functor[F]): F[B] =
    functor.map(src)(func)
}
```

因此，若调用 `map` 的对象类型中没有定义 `map` 函数，如：

```Scala
XXX.map(x => x + 1)
```

则编译器将使用 `FunctorOps` 进行隐式类型转换：

```Scala
new FunctorOps(XXX).map(x => x + 1)
```

而 `FunctorOps.map` 函数需要一个 `implicit` 参数，因此编译器将寻找合适类型的 `Functor` 实例，并插入：

```Scala
new FunctorOps(XXX).map(x => x + 1)(functor instance)
```

此时，可成功执行。

## 2.5.3 Instances for Custom Types

一般只要实现 `Functor.map` 函数即可定义一个 `Functor` 实例：

```Scala
implicit val optionFunctor: Functor[Option] =
  new Functor[Option] {
    def map[A, B](value: Option[A])(func: A => B): Option[B] =
      value.map(func)
  }
```

但有时在创建 instances 的时候，需要 **注入依赖**，例如，若要实现 `Functor[Future]` 实例，按照上面的套路应该这么写：

```Scala
implicit val futureFunctor: Functor[Future] =
  new Functor[Future] {
    override def map[A, B](fa: Future[A])(f: A ⇒ B): Future[B] =
      fa.map(f)
  }
```

但是，注意 `Future.map` 需要两个参数：

```Scala
def map[S](f: T => S)(implicit executor: ExecutionContext): Future[S] = transform(_ map f)
```

因此上面的定义将编译出错，原因是作用域中缺少 `ExecutionContext`，解决方式是在创建 `futureFunctor` 的时候添加 `implicit ExecutionContext` 参数：

```Scala
implicit def futureFunctor(implicit ec: ExecutionContext): Functor[Future] =
  new Functor[Future] {
    override def map[A, B](fa: Future[A])(f: A ⇒ B): Future[B] =
      fa.map(f)
  }
```
* `val` -> `def`

当使用 `Functor.apply` 查找 `Functor` 实例时，会展开如下：

```Scala
// 1. 查找 Functor[Future] 实例
Functor[Future]

// 2. 编译器在 implicit 作用域中找到 futureFunctor，并插入函数调用
Functor[Future](futureFunctor)

// 3. futureFunctor 是一个函数，需要一个 implicit ExecutionContext 参数，编译器继续查找 ExecutionContext 实例，找到后插入
Functor[Future](futureFunctor(executionContext))
```

最后，在 `futureFunctor` 函数内部调用 `future.map` 时，就已经有 `ExecutionContext` 了，自然成功执行。

## 3.5.4 Exercise: Branching out with Functors

为 `Tree` 实现 `Functor[Tree]` 实例，`Tree` 定义如下：

```Scala
sealed trait Tree[+A]

final case class Branch[A](left: Tree[A], right: Tree[A]) extends Tree[A]
final case class Leaf[A](value: A) extends Tree[A]
```

实现 `Functor[Tree]`：

```Scala
object Tree {
  implicit val treeFunctor: Functor[Tree] =
    new Functor[Tree] {
      override def map[A, B](fa: Tree[A])(f: A ⇒ B): Tree[B] =
        fa match {
          case Branch(l, r) ⇒ Branch(map(l)(f), map(r)(f))
          case Leaf(v)      ⇒ Leaf(f(v))
        }
    }
}
```

使用 `Functor[Tree]` 实例：

```Scala
import cats.Functor
import cats.syntax.functor._

val t: Tree[Int] = Branch(Branch(Leaf(1), Leaf(2)), Leaf(3))

// 1.
Functor[Tree].map(t)(_ * 2)  // Branch(Branch(Leaf(2),Leaf(4)),Leaf(6))

// 2. 
t map (_ + 1)  // Branch(Branch(Leaf(2),Leaf(3)),Leaf(4))
```