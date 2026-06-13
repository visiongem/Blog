---
title: "记录 Kotlin 里 Unit 和 Nothing 的区别"
date: 2026-06-05T10:00:00+08:00
draft: false
categories: ["Android"]
tags: ["Android", "Kotlin"]
---
![封面](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/nothing.png)

写 Kotlin 久了，Function 签名里写 `Unit` 已经成了肌肉记忆，偶尔看到源码里冒出个 `Nothing` 也没深究——反正能跑就行。直到有一次定义泛型回调，发现 Java 那边 `void` 的坑 Kotlin 居然用 `Unit` 填得干干净净，而 `TODO()` 背后藏着的 `Nothing` 又打开了类型系统的另一扇门。整理一下。

## 一、先搞清楚问题：Java 的 void 为什么难受

Java 里 `void` 不是类型，它只是个关键字，表示"这个方法不返回任何值"。这带来两个麻烦：

1. **泛型不兼容** —— 如果想写 `Callback<void>`，不行，得分开维护两套接口：一套有返回值的 `Callback<T>`，一套没返回值的 `VoidCallback`。
2. **函数式编程别扭** —— `Function<T, R>` 必须有返回类型，没有返回值的场景只能用 `Consumer<T>`、`Runnable` 这些"补丁"接口来兜底。

Kotlin 的解法很直接：**把"没有返回值"这件事也变成一个类型**。

## 二、Unit：告诉你"我回来了，但没带东西"

### 定义

`Unit` 在 Kotlin 源码里长这样：

```kotlin
public object Unit {
    override fun toString() = "kotlin.Unit"
}
```

一个 `object` 单例。`Unit` 是一个**有且仅有一个实例**的类型。这一点和 Java 的 `void` 完全不同——`void` 不能当类型用，`Unit` 可以。

### 基础用法

```kotlin
// 这两种写法完全等价
fun printLog(): Unit {
    println("log printed")
}

fun printLog() {
    println("log printed")  // 省略 Unit，编译器自动补上
}
```

Kotlin 里不写返回类型的函数，编译器默认返回类型就是 `Unit`。

### 真正的威力：泛型场景

```kotlin
// 一个通用的任务执行器
interface Task<T> {
    fun execute(): T
}

// Java 里还需要单独写个 VoidTask 接口
// Kotlin 里直接传 Unit 就行
class LogTask : Task<Unit> {
    override fun execute() {
        println("任务完成")  // 隐式返回 Unit
    }
}

// Lambda 场景更常见
val onClick: (View) -> Unit = { view ->
    println("clicked: ${view.id}")
}

val onSuccess: () -> Unit = {
    println("成功了")
}
```

因为 `Unit` 是真实类型，它可以直接填入泛型参数，不需要任何"特供版"接口。

### 和 Java 互操作

| 场景 | Kotlin | Java 看它 |
|------|--------|----------|
| 函数返回类型 | `Unit` | `void` |
| Lambda 返回类型 | `(Int) -> Unit` | SAM 接口的 `return;` |
| 参数位置 | `Deferred<Unit>` | `Deferred<Unit>`（不能写 void）|

**补充一个细节**：Kotlin 的 Lambda 返回 `Unit` 时，如果在 Java 侧被当作 SAM（Single Abstract Method）接口使用，返回的其实是 `Unit` 单例对象，Java 侧拿到的就是 `kotlin.Unit.INSTANCE`。

### Unit 小结

| 要点 | 说明 |
|------|------|
| 是什么 | 单例 object，唯一实例 `Unit` |
| 对应 Java | 语义上 ≈  `void`，但它是真正类型 |
| 可以当泛型参数 | ✅ |
| 能不能创建多个实例 | ❌ 单例 |
| 典型场景 | 回调、无返回值的 Task、协程 `LaunchedEffect` 的 block |

## 三、Nothing：这个函数永远不回来了

### 定义

`Nothing` 在 Kotlin 源码里是这样：

```kotlin
public class Nothing private constructor()
```

注意 **`private constructor()`**——外部无法实例化。`Nothing` 是一个**没有任何实例**的类型。

如果说 `Unit` 是"回来了但空着手"，那 `Nothing` 就是**根本没打算回来**。

### 基础用法：永不返回的函数

```kotlin
// 抛出异常的函数，返回 Nothing
fun fail(message: String): Nothing {
    throw IllegalStateException(message)
}

// 无限循环的函数，也返回 Nothing
fun runForever(): Nothing {
    while (true) {
        // 永远不会 return
    }
}
```

实际开发中最常见的 `Nothing` 出现在这两个标准库函数里：

```kotlin
// TODO()——占位符，调用就抛异常
public inline fun TODO(): Nothing = throw NotImplementedError()

// error()——主动抛异常
public inline fun error(message: Any): Nothing = throw IllegalStateException(message.toString())
```

写了 `fun getUser(): User = TODO()` 之后代码能编译通过，就是因为 `TODO()` 返回 `Nothing`，而 `Nothing` 是**所有类型的子类型**——这是理解 `Nothing` 最关键的一点。

### 类型系统的"底类型"（Bottom Type）

`Nothing` 的地位可以这样理解：

```
        Any?
         ↑
        Any
       ↗   ↖
   String    Int    ...  （所有具体类型）
       ↖   ↗       ↗
      Nothing    Nothing   ← Nothing 是所有类型的子类型
```

因为 `Nothing` 是**一切类型的子类型**，它会带来两个很实用的效果：

#### 效果一：泛型推断不出时的"默认填空"

```kotlin
// emptyList() 的定义是 fun <T> emptyList(): List<T>
// 但调用时不指定类型参数，编译器推断 T = Nothing
val empty = emptyList()       // 推断类型为 List<Nothing>

// 但因为 Nothing 是所有类型的子类型
// List<Nothing> 可以直接赋值给 List<String>
val strings: List<String> = emptyList()  // ✅ 合法

// 同理，failure() 推断 T = Nothing
val result: Result<Int> = Result.failure(exception) // Result<Nothing> → Result<Int> ✅
```

这种设计让空集合、失败状态的创建既类型安全又自然——不用写 `emptyList<String>()` 就能让编译器自己推出来。

#### 效果二：智能类型推断与死代码检测

```kotlin
fun process(input: String?) {
    val value = input ?: return  // `return` 的类型是 Nothing
    // 编译器知道：走到这里，input 绝不可能是 null
    println(value.length)  // value 自动变为 String（非 null）
}

fun exhaustiveCheck(value: SealedClass): String = when (value) {
    is TypeA -> "A"
    is TypeB -> "B"
    // 如果 SealedClass 有新的子类 TypeC，这行会编译报错
}
```

`?:` 右边的 `return`、`throw` 表达式类型都是 `Nothing`，编译器看到 `Nothing` 就知道这个分支不会正常执行完毕，从而在后续代码中自动排除掉已处理的情况。

### Nothing 小结

| 要点 | 说明 |
|------|------|
| 是什么 | 没有任何实例的类型，构造器私有 |
| 类型地位 | 所有类型的子类型（Bottom Type） |
| 能实例化吗 | ❌ 不能 |
| 典型场景 | `TODO()`、`error()`、泛型空集合推断、智能类型收窄 |
| 和 `Unit` 的核心区别 | `Unit` = 返回了但没信息；`Nothing` = 根本不会正常返回 |

## 四、一图对比

| 维度 | Unit | Nothing |
|------|------|---------|
| 有实例吗 | ✅ 有，单例 `Unit` | ❌ 没有 |
| 语义 | 正常返回，但没有有意义的返回值 | 从不正常返回（抛异常 / 死循环） |
| 类型层级 | 和 `Any` 平级，是独立类型 | 所有类型的子类型 |
| 泛型兼容 | 填补 `void` 不能用泛型的坑 | 让空集合、失败状态推断自然成立 |
| 常见出现位置 | 函数返回值（默认）、Lambda 参数 | `TODO()`、`error()`、`return` / `throw` 表达式 |
| 编译后 | 多数情况映射为 `void` | 不单独映射，取决于表达式上下文 |

## 五、一个帮助记忆的类比

- **Unit** ≈ 快递小哥敲门，开门签了字，包裹里没东西——**流程走完了，但结果是空的**。
- **Nothing** ≈ 快递小哥送货途中把车开进了黑洞——**永远到不了收货地址，后面的事都不用等了**。

---

**一句话总结**：`Unit` 让 Kotlin 的类型系统不再需要 `void` 关键字，`Nothing` 让类型系统有能力表达"永不返回"这件事，二者一起让泛型、Lambda、异常处理比 Java 干净了一个层次。

`Unit` 平时写业务几乎天天用而浑然不觉，`Nothing` 大部分时候在标准库和框架源码里默默干活——但搞明白它们，算是把 Kotlin 类型系统的拼图补上了挺关键的一块。

---

*本文内容基于 Kotlin 标准库源码（`kotlin.Unit`、`kotlin.Nothing`）整理，参考了 Kotlin 官方文档类型系统部分。如有错误欢迎指正。*

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**
