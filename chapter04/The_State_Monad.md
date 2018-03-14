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

后缀表达式很简单，即：

```Scala
1 2 * 2 +
```

后缀表达式对人类比较难理解，但程序却很容易处理它们：

* 从左到右遍历输入字符串，并携带一个 oprand 栈：
  + 若当前字符为数字，则入栈
  + 若当前字符为 operator，则从 stack 弹出两个 operand，计算它们，并将结果入栈
  
 使用 `State` 实现一个后缀表达式的解释器，输入 `1 2 + 2 *` 这样的字符串，可以计算其结果。
 
 思路：将 **单个字符** 解析为一个 `State` 实例，该实例代表 **stack 的转换 + 中间结果**，使用 `flatMap` 连接 `State` 实例，从而实现对任意 **符序列** 的解析。
 
 ### 1. 计算单个字符
 
 实现 `evalOne` 函数，用于解析单个字符：
 
 ```Scala
 import cats.data.State

type CalcState[A] = State[List[Int], A]

def operand(n: Int): CalcState[Int] =
  State {
    stack ⇒ (n :: stack, n)
  }

def operator(op: (Int, Int) ⇒ Int): CalcState[Int] =
  State {
    case a :: b :: tl ⇒ val ans = op(a, b); (ans :: tl, ans)
    case _            ⇒ sys.error("Fail!")
  }

def evalOne(sym: String): CalcState[Int] =
  sym match {
    case "+"  ⇒ operator(_ + _)
    case "-"  ⇒ operator(_ - _)
    case "*"  ⇒ operator(_ * _)
    case "/"  ⇒ operator(_ / _)
    case x    ⇒ operand(x.toInt)
  }
 ```

使用 `evalOne` 可以计算单个字符组成的表达式，例如：

```Scala
evalOne("10").run(Nil).value  // (List(10), 10)
evalOne("+").run(Nil).value  // 运行时异常 Fail!
```

使用 `evalOne` 时，需要：

* 提供 `Nil` 作为 initial state
* 只能计算数值，如果传入 `+` 则抛异常

## 2. 计算字符序列

借助 `evalOne` `map` 和 `flatMap` 可以实现更复杂序列的计算：

```Scala
val program: CalcState[Int] =
  for {
    _ ← evalOne("111")
    _ ← evalOne("6")
    x ← evalOne("*")
  } yield x

// res0: (List[Int], Int) = (List(666),666)
program.run(Nil).value
```
* 大部分计算都 **发生在 stack** 上，因此忽略 `evalOne("111")` 和 `evalOne("6")` 的 **中间结果**

将上面的代码泛化，实现 `evalAll`，实现对 `List[String]` 的计算：

```Scala
import cats.data.State
import cats.syntax.applicative._

def evalAll(input: List[String]): CalcState[Int] =
  input.foldLeft(0.pure[CalcState])((state, x) ⇒ {
    for {
      _ ← state  // 未用到 state 的计算结果
      b ← evalOne(x)
    } yield b
  })
  
def evalAll2(input: List[String]): CalcState[Int] =
  input.foldLeft(0.pure[CalcState])((state, x) ⇒ state.flatMap(_ ⇒ evalOne(x)))  // // 未用到 state 的计算结果
```

`evalAll` 使用如下：

```Scala
// res0: (List[Int], Int) = (List(3),3)
evalAll(List("1", "2", "+")).run(Nil).value
// res1: (List[Int], Int) = (List(9),9)
evalAll2(List("1", "2", "+", "3", "*")).run(Nil).value
```

### 3. 计算字符串

因为 `evalOne` 和 `evalAll` 都返回 `State`，因此可以用 `map` 和 `flatMap` 任意组合它们：

```Scala
val f =
  for {
    _ ← evalAll(List("1", "5", "+"))
    _ ← evalAll(List("110", "1", "+"))
    x ← evalOne("*")
  } yield x

// res3: (List[Int], Int) = (List(666),666)
f.run(Nil).value
```

本练习最后以实现 `evalInput` 函数结束，该函数接受 `String` 格式的后缀表达式，返回其计算结果：

```Scala
def evalInput(input: String): Int =
  evalAll(input.split(" ").toList).runA(Nil).value

evalInput("1 2 + 3 *")  // 9
```
