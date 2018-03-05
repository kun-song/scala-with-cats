# 1.6 Controlling Instance Selection

使用 type class，需要 Scala 编译器自动选择 type class instances，而 instance selection 受以下两个因素影响：

* 在类型、子类上分别定义的两个 instances 之间有什么关系？
  + 例如，假设有实例 `JsonWriter[Option[Int]]`，则表达式 `Json.toJson(Some(1))` 是否会选择使用该实例？（`Some` 是 `Option` 的子类）
* 当有多个相同类型的 type class instances 时，应如何选择？
  + 例如，若定义有多个 `JsonWriter[Person]`，则 `Json.toJson(aPerson)` 会选择哪个？
  
## 1.6.1 Variance

在定义 type class 的时候，可以为 type parameter 添加 variance annotation，以影响 type class 的变形，以及 implicit resolution。

variance 与子类型相关，若任何需要类型 `A` 的地方，都可以使用类型 `B` 的值替代，则称 `B` 是 `A` 的子类。

当涉及 type constructor 的时候，有协变、逆变的概念。

### 协变

协变指：若 `B` 是 `A` 的子类，则 `F[B]` 也是 `F[A]` 的子类。

协变通过 `+` 符号表示：

```Scala
trait F[+A]
```

协变可以对很多数据类型建模，例如 `List` 和 `Option`。

```Scala
trait List[+A]
trait Option[+A]
```

通过协变，可以使用子类的集合替代父类的集合，这在 Java 中是不允许的。

### 逆变

逆变指：若 `B` 是 `A` 的子类，则 `F[A]` 是 `F[B]` 的子类。

逆变通过 `-` 符号表示：

```Scala
trait F[-A]
```

协变可用于对表示 **处理** 的类型建模，例如 `JsonWriter` type class：

```Scala
trait JsonWriter[-A] {
  def write(value: A): Json
}
```

假设有两个类型 `Shape` 和 `Circle`，以及对应的两个 `JsonWriter` instance：

```Scala
sealed trait Shape
case class Circle(r: Double) extends Shape

val shape: Shape = Circle(1)
val circle: Circle = Circle(2)

val shapeWriter: JsonWriter[Shape] =
  _ => JsString("a shape")
  
val circleWriter: JsonWriter[Circle] =
  circle => JsObject(Map("name" → JsString("circle"),
    "r = " → JsNumber(circle.r)))
```

以及函数：

```Scala
def format[A](v: A, writer: JsonWriter[A]): Json = writer.write(v)
```

问题来了，`shape` `circle` 和 `shapeWriter` `circleWriter` 两两组合，哪些组合可以合法调用 `format` 函数呢？

分析一下，首先 `JsonWriter` 为逆变，而 `Circle` 是 `Shape` 的子类，所以 `JsonWriter[Shape]` 是 `JsonWriter[Circle]` 的子类。

若使用 `circle` 作为第一个参数，则 `format` 中的类型参数变成 `Circle`，所以第二个参数类型为 `JsonWriter[Circle]`，第二个参数可以用其子类，即 `JsonWriter[Shape]` 替代，所以以下两种调用都是合法的：

```Scala
format(circle, circleWriter)
format(circle, shapeWriter)
```

若使用 `shape` 作为第一个参数，则 `format` 中的类型参数变成 `Shape`，所以第二个参数类型为 `JsonWriter[Shape]`，此时不能使用 `JsonWriter[Shape]` 调用，因为这是父类，所以只有一种合法调用：

```Scala
format(shape, shapeWriter)
```

### 不变

不变是最简单的情况，没有 `+` 和 `-` 即为不变：

```Scala
trait F[A]
```

不变意味着，无论 `A` 和 `B` 有什么父子类关系，`F[A]` 和 `F[B]` 都没有任何父子关系。

### 总结

还有两个重要问题，假设：

```Scala
sealed trait A
final case object B extends A
final case object C extends A
```

问题来了：

* 定义在父类上的 instance 是否会被选择？
  + 例如，为 `A` 定义 instance，该 instance 是否可用于 `B` 或 `c` 类型的值
* 定义在子类上的 instance 是否比定义在父类上的 instance **优先级高**
  + 例如，`A` 和 `B` 都定义了 instance，则使用 `B` 类型的 **值** 时，`B` 的 instance 是否被优先选择

实际上，这两者 **不可兼得**，在不同的变型下，可以获取不同的效果：

|      Type Class 变型         |        不变        |         协变        |        逆变        |
|:----------------------------:|:-----------------:|:-------------------:|:------------------:|
|   Supertype instance used?   |         No        |          No         |        Yes         |
| More specific type preferred?|         No        |          Yes        |        No          |

世界上没有完美的系统，不可兼得也是清理之中，3 中变型根据个人喜好选择即可，而 Cats 倾向于使用 invariance，这意味着，若有值 `Some(1)`，则针对 `Option` 的 instance 不会被使用，这在前面已经展示过了，可以通过加类型修饰 `Some(1): Option[Int]` 或使用 `Option(1)` 等方式解决。










