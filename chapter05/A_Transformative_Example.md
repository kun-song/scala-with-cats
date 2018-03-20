# 5.2 A Transformative Example

Cats 为很多 monad 提供了 transformer，transformer 与 monad 同名，但多一个 `T` 后缀：

* `EitherT` 用于组合 `Either` 和其他 monad
* `OptionT` 用于组合 `Option` 和其他 monad

假设现在有一个 `List[Option[Int]]`：

```Scala
/**
  * List(Some(1), Some(2), None)
  */
val xs = List(Some(1), Some(2), None)
```

若要对每个 `Int` 加 1，需要使用嵌套的 `map`：

```Scala
/**
  * List(Some(2), Some(3), None)
  */
xs.map { option ⇒ {
  option.map { x ⇒
    x + 1
  }
}}
```

下面用 `OptionT` 来组合 `List` 和 `Option`，以消除上面的嵌套 `map`，`List[Option[Int]]` 是嵌套的 monad，外层 monad 是 `List`，内层 monad 是 `Option`，因此使用 `OptionT`，且将 `List` 作为其第一个类型参数，即 `OptionT[List, A]`：

```Scala
import cats.data.OptionT

type ListOption[A] = OptionT[List, A]
```
* 使用类型别名，将 `List[Option]` 这个嵌套的 monad 转换为一个 monad，方便使用

使用 `OptionT` 构造器或 `pure` 创建 `ListOption` 实例：

```Scala
import cats.data.OptionT
import cats.instances.list._  // for monad
import cats.syntax.applicative._  // for pure

type ListOption[A] = OptionT[List, A]

// a: ListOption[Int] = OptionT(List(Some(10)))
// b: ListOption[Int] = OptionT(List(Some(32)))
val a: ListOption[Int] = OptionT(List(Option(10)))
val b: ListOption[Int] = 32.pure[ListOption]
```

现在，`map` 不再嵌套：

```Scala
// 使用 OptionT 进行 monad transform
a.map(_ * 10)  // OptionT(List(Some(100)))

// 嵌套 monad
xs.map { option ⇒ {
  option.map { x ⇒
    x + 1
  }
}}
```

`flatMap` 也不再需要嵌套：

```Scala
val a: ListOption[Int] = OptionT(List(Option(10)))
val b: ListOption[Int] = 32.pure[ListOption]

// OptionT(List(Some(42)))
a.flatMap { aa ⇒
  b.map { bb ⇒
    aa + bb
  }
}

val xs = List(Some(1), Some(2), None)
val ys = List(Some(10))

// List(List(11), List(12))
xs.flatMap { ox ⇒ {
  ox.map { x ⇒
    ys.flatMap { oy ⇒ {
      oy.map { y ⇒ {
        x + y
      }}
    }}
  }
}}
```

使用 monad transformer 后，可以将多层嵌套的 monad 视为一个 monad，`map` 和 `flatMap` 直接对 **最内层** 的 monad content 其作用，消除了嵌套。

>Complexity of Imports
>
>例子中的代码是如何 **糅合** 在一起的呢？从 `import` 就可以看出一切。
>
>例子中导入 `cats.syntax.applicative` 以获取 `pure` syntax。
>
>1. `pure` 需要一个 `implicit` `Applicative[ListOption]` 实例做为参数. 所有 `Monad`s 都是 `Applicative`s，所以暂时不区分两者。
>
>2. 为产生 `Applicative[ListOption]` 实例，需要有 `Applicative[List]` 和 `Applicative[OptionT]` 实例：
>  * `OptionT` 是 Cats 定义的类，其实例由其伴生对象提供 
>  * `Applicative[List]` 实例由 `cats.instances.list` 导入
>
>我们没有导入 `cats.syntax.functor` 和 `cats.syntax.flatMap`，因为：
>* `OptionT` 是一个具体的类型，它自己本身就定义有 `map` 和 `flatMap`
>* 即使导入了，也不会有任何影响，编译器将忽略他们
>
>通过 `cats.implicits` 可以一次导入全部需要的内容。
