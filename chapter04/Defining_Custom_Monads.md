# 4.10 Defining Custom Monads

前面介绍了很多 Cats 或 Scala 内置的 monad 实例，我们也可以为 *自己的类型* 定义 monad 实例，只要实现 3 个方法即可：

1. `pure`
2. `flatMap`
3. `tailRecM`

其中 `pure` 和 `flatMap` 是 monad 概念所需要的，而 `tailRecM` 则是 Cats 用来优化 `flatMap` **嵌套调用** 所消耗的栈空间的：

```Scala
import cats.Monad

import scala.annotation.tailrec

val optionMonad: Monad[Option] =
  new Monad[Option] {
    override def pure[A](x: A): Option[A] = Some(x)

    @tailrec
    override def tailRecM[A, B](a: A)(f: A ⇒ Option[Either[A, B]]): Option[B] =
      f(a) match {
        case Some(Right(b)) ⇒ Some(b)
        case Some(Left(a1)) ⇒ tailRecM(a1)(f)
        case None           ⇒ None
      }

    override def flatMap[A, B](fa: Option[A])(f: A ⇒ Option[B]): Option[B] =
      fa match {
        case Some(x)  ⇒ f(x)
        case None     ⇒ None
      }
  }
```

若可以把 `tailRecM` 实现为 **尾递归**，则 Cats 可以保证嵌套调用 `flatMap` 的栈安全，若 `tailRecM` 不是尾递归，则嵌套太多时可能发生 `StackOverflowError`，Cats 中定义的 Monad 实例都是尾递归的，因此可以随意嵌套。

## 4.10.1 Exercise: Branching out Further with Monads

为前一章中的 `Tree` 定义 `Monad[Tree]` 实例，`Tree` 定义如下：

```Scala
sealed trait Tree[A]

final case class Branch[A](left: Tree[A], right: Tree[A]) extends Tree[A]
final case class Leaf[A](value: A) extends Tree[A]

object Tree {
  def branch[A](left: Tree[A], right: Tree[A]): Tree[A] = Branch(left, right)
  def leaf[A](value: A): Tree[A] = Leaf(value)
}
```

`Monad[Tree]` 实例实现如下，这里 `tailRecM` 不是尾递归实现，如果要尾递归，非常复杂，为方便使用，将其放在 `Tree` 对象中，并用 `implicit` 修饰：

```Scala
import cats.Monad

object Tree {
  def branch[A](left: Tree[A], right: Tree[A]): Tree[A] = Branch(left, right)
  def leaf[A](value: A): Tree[A] = Leaf(value)

  implicit val treeMonad: Monad[Tree] =
    new Monad[Tree] {
      override def pure[A](x: A): Tree[A] = Leaf(x)

      override def tailRecM[A, B](a: A)(f: A ⇒ Tree[Either[A, B]]): Tree[B] =
        f(a) match {
          case Branch(l, r)   ⇒
            Branch(
              flatMap(l) {
                case Left(a1) ⇒ tailRecM(a1)(f)
                case Right(b) ⇒ pure(b)
              },
              flatMap(r) {
                case Left(a2) ⇒ tailRecM(a2)(f)
                case Right(b) ⇒ pure(b)
              })
          case Leaf(Left(a3)) ⇒ tailRecM(a3)(f)
          case Leaf(Right(b)) ⇒ Leaf(b)
        }

      override def flatMap[A, B](fa: Tree[A])(f: A ⇒ Tree[B]): Tree[B] =
        fa match {
          case Branch(l, r) ⇒ Branch(flatMap(l)(f), flatMap(r)(f))
          case Leaf(x)      ⇒ f(x)
        }
    }
}
```

有了 `Monad[Tree]` 实例，就可以在 `Tree` 上使用 `map` 和 `flatMap` 了：

```Scala
import cats.syntax.functor._
import cats.syntax.flatMap._

/**
  * t1: Tree[Int] = Branch(Branch(Leaf(1),Leaf(2)),Leaf(3))
  * t2: Tree[Int] = Branch(Branch(Leaf(2),Leaf(3)),Leaf(4))
  * t3: Tree[Int] = Branch(Branch(Branch(Leaf(2),Leaf(4)),Branch(Leaf(3),Leaf(6))),Branch(Leaf(4),Leaf(8)))
  */
val t1 = Tree.branch(Tree.branch(Tree.leaf(1), Tree.leaf(2)), Tree.leaf(3))
val t2 = t1.map(_ + 1)
val t3 = t2.flatMap(x ⇒ Tree.branch(Tree.leaf(x), Tree.leaf(x * 2)))
```

>**注意**
>
>直接使用 `Branch` 和 `Leaf` 函数的返回类型分别是 `Branch` 和 `Leaf`，而非 `Tree`，但是我们定义的是 `Monad[Tree]`，因此用 `Branch` 和 `Leaf` 构造的结果是无法使用 `Monad[Tree]` 的（无法使用其 `map` 和 `flatMap` 方法），因此在 `Tree` 对象中定义了 smart constructor `branch` 和 `leaf`，它们与直接用 case class constructor 的区别仅仅是 **返回值类型**。

有了 `Monad[Tree]` 实例就有了 `map` 和 `flatMap`，有了 `map` 和 `flatMap` 就可以使用 for 解析：

```Scala
/**
  * Branch(Branch(Branch(Leaf(10),Leaf(100)),Leaf(1000)),Branch(Branch(Leaf(20),Leaf(200)),Leaf(2000)))
  */
for {
  a ← Tree.leaf(1)
  b ← Tree.branch(Tree.leaf(a), Tree.leaf(a * 2))
  c ← Tree.branch(Tree.branch(Tree.leaf(b * 10), Tree.leaf(b * 100)), Tree.leaf(b * 1000))
} yield c
```
* 在 for 解析（`flatMap`）中，`Tree` 可能会不断 **长大**！

>* The monad for `Option` provides **fail-fast** semantics.
>* The monad for `List` provides **concatenation** semantics.
>
>What are the semantics of `flatMap` for a binary tree?
>
>Every node in the tree has the potential to be replaced with a whole subtree, producing a kind of **growing** or **feathering** behaviour, reminiscent of list concatenation along two axes.