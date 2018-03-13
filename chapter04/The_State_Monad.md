# 4.9 The State Monad

`cats.data.State` 可以携带 state + computation。

将 `State` 实例定义为 atomic state operation，使用 `map` 和 `flatMap` 将它们串起来，从而以纯函数方式模拟出 **mutable state**。

## 4.9.1 Creating and Unpacking `State`

`State[S, A]` 实例其实代表一个 `S => (S, A)` 函数，其中 `S` 为 state 的类型，`A` 为 result 的类型：

```Scala
import cats.data.State

val a = State[Int, String] {
  state ⇒ (state, s"The state is $state")
}
```

从例子中可以看出，一个 `State` 实例会做两件事：

* state transformation
  + 将 input state 转换为 output state
* compute a result

通过提供 initial state，可以运行 `State` 实例：

* `State.run` 返回 (state, result)
* `State.runS` 返回 state
* `State.runA` 返回 result

以上 3 个函数的返回值类型都是 `Eval`，从而保证栈安全：

```Scala
/**
  * state: Int = 10
  * result: String = The state is 10
  */
val (state, result) = a.run(10).value

/**
  * state1: Int = 10
  */
val state1 = a.runS(10).value

/**
  * result1: String = The state is 10
  */
val result1 = a.runA(10).value
```

## 4.9.2 Composing and Transforming `State`

与 `Reader` 和 `Writer` 类似，`State` 强大之处也是其 **组合性**，每个 `State` 实例代表一个 atomic state transformation，使用 `map` 和 `flatMap` 组合 `State` 实例后，组合的结果代表 a complete sequence of transformations：

```Scala
import cats.data.State

val step1 = State[Int, String] {
  state ⇒ {
    val ans = state + 1
    (ans, s"Result of step1: $ans")
  }
}

val step2 = State[Int, String] {
  state ⇒ {
    val ans = state * 2
    (ans, s"Result of step2: $ans")
  }
}

val step3 = State[Int, String] {
  state ⇒ {
    val ans = state * 3
    (ans, s"Result of step3: $ans")
  }
}

val both =
  for {
    a ← step1
    b ← step2
    c ← step3
  } yield (a, b, c)

/**
  * res0: (Int, (String, String, String)) = (666,(Result of step1: 111,Result of step2: 222,Result of step3: 666))
  */
both.run(110).value
```

上面例子中：

* state
  + 虽然 for 解析中没有显示写出 **state 相关操作**，但最终状态依然是 **按照顺序** 依次执行各个 `State` 实例的 state transformation 的结果：
    1. inital state = 110
    2. step1 转换后，state = 110 + 1 = 111
    3. step2 转换后，state = 111 * 2 = 222
    4. step3 转换后，state = 222 * 3 = 666
* result
  + 按照 `yield` 中 `(a, b, c)` 方式计算得出。
  + 若 `yield a + b + c` 则最后结果为 `(Int, String)`

使用 `State` 套路如下：

* 将每步计算表示为 `State` 实例
* 使用标准 monad 操作符 **组合** `State` 实例

Cats 提供了一些构造器，用于创建 primitive step：

* `get`: extracts the **state** as the **result**
* `set`: update the **state**, return unit as **result**
* `pure`: ignore the **state**, return a supplied **result**
* `inspect`: extract the **state** via a transformation function
* `modify`: modify the **state** using an update function

```Scala
val getDemo: State[Int, Int] = State.get[Int]
getDemo.run(10).value  // res0: (Int, Int) = (10,10)

val setDemo: State[Int, Unit] = State.set[Int](30)
/**
  * 虽然这里 initial state = 10，但结果还是 (30, ())
  */
setDemo.run(10).value  // res1: (Int, Unit) = (30,())

val pureDemo: State[Int, String] = State.pure[Int, String]("xxx")
pureDemo.run(10).value  // res0: (Int, String) = (10,xxx)

val inspectDemo: State[Int, String] = State.inspect[Int, String](_ + "!!!")
inspectDemo.run(10).value  // res0: (Int, String) = (10,10!!!)

val modifyDemo: State[Int, Unit] = State.modify[Int](_ * 6)
modifyDemo.run(111).value  // res0: (Int, Unit) = (666,())
```

可用 for 解析组合以上 building block，且一般 **忽略** 仅对 state 做 **转换** 的中间步骤：

```Scala
import cats.data.State
import State._

val program: State[Int, (Int, Int, Int)] =
  for {
    a ← get[Int]
    _ ← set[Int](a + 1)
    b ← get[Int]
    _ ← modify[Int](_ + 1)
    c ← inspect[Int, Int](_ * 1000)
  } yield (a, b, c)

program.run(1).value  // res0: (Int, (Int, Int, Int)) = (3,(1,2,3000))
```

## 4.9.3 Exercise: Post-Order Calculator
