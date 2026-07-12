---
title: "Kotlin Flow 截断四兄弟：drop、dropWhile、take、takeWhile"
date: 2026-06-23T23:00:00+08:00
draft: false
categories: ["Android"]
tags: ["Android", "Kotlin", "协程"]
---

`filter` 是「挑出符合条件的」，每条数据过一遍筛子。`drop` 和 `take` 干的不是同一件事——它们对**位置**和**边界**敏感：丢弃头几条、只要头几条、从某个条件之后开始放行、到某个条件之前为止。

这四个操作符都是**截断型**的：持续处理数据，碰到某个边界之后行为翻转。分成两组看：

- `drop` / `dropWhile`：**前半段扔掉**，后半段放行
- `take` / `takeWhile`：**只要前半段**，后半段扔掉

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/flowdrop0.png)

## 一、先记住一句话

这四个操作符一旦「过界」，**不会再回头判断**：

- `drop(n)` 数够了 n 条之后，后面全部放行，不会再看后面的数据长什么样
- `dropWhile { }` 条件第一次为 false 之后，后面全部放行，不会再回头判断条件
- `take(n)` 拿够 n 条之后，直接结束，后面的数据不执行
- `takeWhile { }` 条件第一次为 false 时，直接结束，**包括这条不满足条件的数据也不会放行**

理解这个「过界不回」的机制，比记住每个函数签名更有用。

## 二、drop：跳过前 N 条


```
/**
 * Returns a flow that ignores first [count] elements.
 * Throws [IllegalArgumentException] if [count] is negative.
 */
public fun <T> Flow<T>.drop(count: Int): Flow<T> {
    require(count >= 0) { "Drop count should be non-negative, but had $count" }
    return flow {
        var skipped = 0
        collect { value ->
            if (skipped >= count) emit(value) else ++skipped
        }
    }
}
```


```kotlin
flow { emit(1); emit(2); emit(3); emit(4); emit(5) }
    .drop(2)
    .collect { println(it) }

// 输出：3  4  5
// 前 2 条被丢了，后面的全部放行
```

`drop(2)` 内部维护一个计数器。前 2 次 `emit` 被吞掉，第 3 次开始，计数器到上限了，之后所有数据原样放行。

**实际场景：列表里跳过置顶项。** 后端返回的数据前两条是置顶内容，已经在 UI 上单独渲染了，展示普通列表时跳过它们：

```kotlin
fun observeNormalPosts(): Flow<Post> = repository.observePosts()
    .drop(2)  // 跳过两条置顶，后面的直接展示
```

`drop(0)` 等于什么都不做，所有数据放行。这是合法的——边界情况处理起来不需要额外判断。

## 三、dropWhile：条件为真就一直扔


```
/**
 * Returns a flow containing all elements except first elements that satisfy the given predicate.
 */
public fun <T> Flow<T>.dropWhile(predicate: suspend (T) -> Boolean): Flow<T> = flow {
    var matched = false
    collect { value ->
        if (matched) {
            emit(value)
        } else if (!predicate(value)) {
            matched = true
            emit(value)
        }
    }
}
```


```kotlin
flow { emit(2); emit(4); emit(5); emit(6); emit(8) }
    .dropWhile { it % 2 == 0 }
    .collect { println(it) }

// 输出：5  6  8
// 2 和 4 偶数 → 丢；5 奇数 → 条件第一次为 false，放行 5，之后不管了直接放
```

**关键行为：** 条件第一次为 `false` 之后，`dropWhile` 就不再调用 predicate 了。条件碰到的第一个不满足的元素**也会放行**（上面的 5 放出来了）。

再看一个条件始终没被打破的例子：

```kotlin
flow { emit(2); emit(4); emit(6) }
    .dropWhile { it % 2 == 0 }
    .collect { println(it) }

// 输出：什么都没打
// 所有元素都满足条件 → 全部丢 → Flow 空
```

**实际场景：跳过加载状态。** Flow 开头会先 emit 一个 `Loading`，之后才是真正的数据。展示时不需要 `Loading` 占位：

```kotlin
sealed class UiState {
    object Loading : UiState()
    data class Content(val items: List<Item>) : UiState()
}

fun observeContentOnly(): Flow<UiState> = stateFlow
    .dropWhile { it is UiState.Loading }  // Loading 及之前的数据全丢，Content 开始放
```

一旦第一条 `Content` 出现，后面再也不会因为 `Loading` 丢掉数据——`dropWhile` 只判断到第一个不满足条件的元素为止。

## 四、take：只要前 N 条


```

/**
 * Returns a flow that contains first [count] elements.
 * When [count] elements are consumed, the original flow is cancelled.
 * Throws [IllegalArgumentException] if [count] is not positive.
 */
public fun <T> Flow<T>.take(count: Int): Flow<T> {
    require(count > 0) { "Requested element count $count should be positive" }
    return flow {
        val ownershipMarker = Any()
        var consumed = 0
        try {
            collect { value ->
                // Note: this for take is not written via collectWhile on purpose.
                // It checks condition first and then makes a tail-call to either emit or emitAbort.
                // This way normal execution does not require a state machine, only a termination (emitAbort).
                // See "TakeBenchmark" for comparision of different approaches.
                if (++consumed < count) {
                    return@collect emit(value)
                } else {
                    return@collect emitAbort(value, ownershipMarker)
                }
            }
        } catch (e: AbortFlowException) {
            e.checkOwnership(owner = ownershipMarker)
        }
    }
}

```

```kotlin
flow { emit(1); emit(2); emit(3); emit(4); emit(5) }
    .take(2)
    .collect { println(it) }

// 输出：1  2
// 拿到 2 条就停，emit(3) 的调用会被 AbortFlowException 中断
```

`take(2)` 内部有一个计数器。拿够 2 条之后，直接抛出 `AbortFlowException`（内部异常，不影响协程），Flow 收集结束。

**这里有一个容易踩的坑：** `take(n)` 触发结束后，`flow { }` 块里后面的代码不会执行。如果里面有清理逻辑，别放在 `emit` 之后——用 `finally`：

```kotlin
flow {
    try {
        emit(1)
        emit(2)
        emit(3)  // take(2) 的话这行不会执行
    } finally {
        cleanup()  // finally 一定会执行
    }
}
.take(2)
.collect { println(it) }
```

`AbortFlowException` 继承自 `CancellationException`，会被协程框架特殊处理，不会进入 `catch` 块——所以 `take` 截断不会污染你的异常处理。

**实际场景：debounce + flatMapLatest 中只取最新结果，旧结果自动丢弃。**
输入框连续打字，debounce 之后发起搜索请求；如果在请求返回前又输入了，`flatMapLatest` 会取消旧的 Flow——而内层 `take(1)` 虽只 emit 一次，但配合外层取消机制，旧请求的 `emit` 还没执行就被取消了：

```kotlin
searchQuery
    .debounce(300)                          // 300ms 内连续输入只留最后一次
    .flatMapLatest { query ->               // 新 query 进来，旧 flow 取消
        flow { emit(api.search(query)) }    // 旧请求可能还没 emit 就被 cancel
            .take(1)                        // 内层只取一个结果
    }
```

这里的 `take(1)` 不是防抖的关键——`flatMapLatest` 的取消才是。但如果不用 `flatMapLatest`、手动 `launch` 收集，`take(1)` 可以保证拿到一个结果就结束收集：

```kotlin
// 在不支持 flatMapLatest 的场景下，手动 launch + take(1) 结束收集
scope.launch {
    repository.observeUpdates()
        .take(1)  // 拿一次就结束，不持续监听
        .collect { handleOnce(it) }
}
```

**实际场景：拿一次就停，不持续监听。** 比如只关心数据库当前快照，不想被后续变更反复触发：

```kotlin
fun loadCurrentSnapshot(): Flow<Data> = repository.observeData()
    .take(1)  // 拿到当前值就结束，数据库后续变更不再触发 collect
```

## 五、takeWhile：条件为真一直拿


```
/**
 * Returns a flow that contains first elements satisfying the given [predicate].
 *
 * Note, that the resulting flow does not contain the element on which the [predicate] returned `false`.
 * See [transformWhile] for a more flexible operator.
 */
public fun <T> Flow<T>.takeWhile(predicate: suspend (T) -> Boolean): Flow<T> = flow {
    // This return is needed to work around a bug in JS BE: KT-39227
    return@flow collectWhile { value ->
        if (predicate(value)) {
            emit(value)
            true
        } else {
            false
        }
    }
}
```


```kotlin
flow { emit(2); emit(4); emit(5); emit(6); emit(8) }
    .takeWhile { it % 2 == 0 }
    .collect { println(it) }

// 输出：2  4
// 2、4 偶数 → 放行；5 奇数 → 条件第一次为 false，结束。5 不放行。
```

**跟 `dropWhile` 的关键区别：** 条件第一次为 `false` 的那个元素，`dropWhile` 会放行，`takeWhile` **不会放行**。

对比最直观：

```kotlin
val data = flow { emit(2); emit(4); emit(5); emit(6); emit(8) }

data.dropWhile { it % 2 == 0 }.collect { print("$it ") }  // 5 6 8
data.takeWhile { it % 2 == 0 }.collect { print("$it ") }  // 2 4
```

同一个序列、同一个条件——`dropWhile` 从 5 开始放，`takeWhile` 到 4 为止。

**实际场景：列表里读到分隔符为止。** 读取一段连续的标题文本，直到遇到空行：

```kotlin
flow {
    emit("标题1")
    emit("标题2")
    emit("")       // 分隔空行，读到这停止
    emit("正文1")   // 不会被收集到
    emit("正文2")
}
.takeWhile { it.isNotEmpty() }
.collect { titles.add(it) }

// titles = ["标题1", "标题2"]
```

## 六、两两对照

### drop vs take

| | `drop(n)` | `take(n)` |
|---|---|---|
| 行为 | 跳过前 n 条，后面的全要 | 只要前 n 条，后面的全停 |
| 元素数量 | N 条之后全部放行 | N 条之后 Flow 结束 |
| 停止方式 | 不停止，只是跳过 | 抛出 AbortFlowException 结束 |
| 类比 | 队列里 pop 掉前几个 | 只 peek 前几个，队列扔掉 |

```kotlin
flow { emit(1); emit(2); emit(3); emit(4); emit(5) }
    .drop(2).collect { print("$it ") }  // 3 4 5
flow { emit(1); emit(2); emit(3); emit(4); emit(5) }
    .take(2).collect { print("$it ") }  // 1 2
```

### dropWhile vs takeWhile

| | `dropWhile { }` | `takeWhile { }` |
|---|---|---|
| 条件真时 | 丢弃 | 放行 |
| 条件第一次假时 | **放行该元素**，之后全放 | **丢弃该元素并结束**，之后全停 |
| 条件永远真 | 全丢，Flow 空 | 全收，Flow 正常结束 |
| 条件永远假 | 全放 | 第一条就停 |

```kotlin
// 同样条件："值小于 5"
val data = flowOf(1, 2, 3, 4, 5, 6, 7)

data.dropWhile { it < 5 }.collect { print("$it ") }  // 5 6 7
// 1-4 丢，5 开始放

data.takeWhile { it < 5 }.collect { print("$it ") }  // 1 2 3 4
// 1-4 收，5 触发结束且不放行
```

## 七、dropWhile / takeWhile 的一个陷阱

条件一旦第一次翻转为 false，predicate 不会再被调用。看这个例子：

```kotlin
flow { emit(2); emit(4); emit(5); emit(4); emit(2) }
    .takeWhile { it % 2 == 0 }
    .collect { print("$it ") }

// 输出：2  4
// 注意：后面还有 4 和 2 也是偶数，但不会再放了
// 因为 5 已经翻过条件，takeWhile 停了
```

这是合理的——`takeWhile` 语义就是「持续拿，直到不满足」。但如果你期待的行为是「拿所有偶数」，那该用的是 `filter`，不是 `takeWhile`：

```kotlin
val data = flow { emit(2); emit(4); emit(5); emit(4); emit(2) }

data.filter { it % 2 == 0 }       // 输出：2  4  4  2  ← 所有偶数
data.takeWhile { it % 2 == 0 }    // 输出：2  4        ← 到第一个奇数就停
data.dropWhile { it % 2 == 0 }    // 输出：5  4  2    ← 第一个奇数之后全放
```

## 八、组合使用

这四个操作符经常叠在一起——先丢掉一些，再取一些：

```kotlin
// 分页：跳过第 1 页的 20 条，拿第 2 页的 20 条
fun loadPage(page: Int, pageSize: Int = 20): Flow<Item> = repository.observeItems()
    .drop((page - 1) * pageSize)  // 跳过前面页的数据
    .take(pageSize)                // 只要本页的数据

// 1, 2...20 → 跳过 0 条，取 20 条
// 21, 22...40 → 跳过 20 条，取 20 条
```

```kotlin
// 跳过配置加载阶段，只取前面 5 条用户记录
fun loadFirstUsers(): Flow<User> = repository.observeUsers()
    .dropWhile { it is User.Config }     // 跳过配置项
    .takeWhile { it is User.NormalUser }  // 拿到普通用户
    .take(5)                               // 只要前 5 条
```

```kotlin
// 示意：跳过前 50ms 的快速发射（防抖后仍有抖动的情况），取第一个稳定值
// predicate 内的时间判断需自行实现，比如依赖每个元素的 timestamp 字段
flowOf(...)
    .dropWhile { it.timestamp < firstEmitTime + 50.milliseconds }
    .take(1)
```

## 九、总结

| 你要的效果 | 用这个 |
|---|---|
| 跳过开头 N 条，后面全要 | `drop(n)` |
| 条件为真时持续跳过，条件破了之后全放 | `dropWhile { }` |
| 只取开头 N 条，后面的完全不要 | `take(n)` |
| 条件为真时持续取，条件破了立刻停 | `takeWhile { }` |
| 挑出所有符合条件的（不是截断） | `filter { }` ← 别跟 takeWhile 搞混 |

这四个操作符跟 `filter` 最大的区别：`filter` 是**逐个判断，独立决策**；`drop`/`take` 是**有状态的，之前的行为影响之后的行为**。`dropWhile` 破了条件之后不回头，`takeWhile` 破了条件之后直接停——脑子里有个「边界」，这四个函数用起来就不会出错。

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**

