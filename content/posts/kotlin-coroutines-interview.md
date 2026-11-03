---
title: "Kotlin 与协程面试 15 问：从 launch、async 到 StateFlow"
date: 2026-11-03T10:00:00+08:00
draft: false
description: "整理 Kotlin 与协程面试常见问题，讲清 launch、async、withContext、lazy、lateinit、异常传播、Flow 与页面生命周期，附代码和面试口述答案。"
categories: ["Android"]
tags: ["Kotlin", "Android", "协程", "Flow", "面试"]
---

整理 Android 面试题时，Kotlin 和协程这一部分很容易出现一种情况：API 天天用，追问两句就开始犹豫。

`launch` 和 `async` 的返回值不一样，这个记得。但 `async` 不调用 `await()`，异常还会不会影响父协程？`lazy` 默认线程安全，那它创建出来的集合能不能随便在多个线程里修改？页面停止收集 Flow，后台的 WebSocket 会不会一起停？

这些问题都需要把使用条件讲清楚。

下面按面试中常见的追问顺序整理了 15 道题，每题包含原理、示例或使用场景，以及一段可以用自己的话复述的答案。涉及属性实现的部分以 **Kotlin/JVM，也就是常见 Android 开发环境**为准。业务代码中的 `api`、`repository` 等表示项目已有依赖，示例省略常规 import。

![Kotlin 与协程面试 15 问封面](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/k0.png)

## 1. 协程是什么？加了 suspend 就会去子线程吗？

协程是一段可以挂起、之后继续执行的计算过程。在 JVM 上，执行代码的仍然是线程；协程可以在等待时挂起，让线程去执行别的任务。参见 [Kotlin 协程基础](https://kotlinlang.org/docs/coroutines-basics.html)。

`suspend` 允许函数调用挂起函数，但它不会自动创建线程，也不保证函数一定发生挂起。

```kotlin
suspend fun calculate(): Long {
    var total = 0L
    repeat(100_000_000) { total += it }
    return total
}
```

在主线程调用这个函数，计算仍然会占住主线程。需要把这类 CPU 密集任务放到合适的调度器：

```kotlin
suspend fun calculate(): Long = withContext(Dispatchers.Default) {
    var total = 0L
    repeat(100_000_000) {
        if (it % 10_000 == 0) ensureActive()
        total += it
    }
    total
}
```

常见选择是：UI 操作用 `Main`，CPU 密集计算用 `Default`，阻塞式 I/O 用 `IO`。对于已经支持非阻塞挂起的 API，要看它自身的线程约定，不必看到网络请求就再包一层 `IO`。Android 官方建议由执行耗时操作的类负责保证函数可以安全地从主线程调用，实际项目也可以注入调度器，方便测试。参见 [Android 协程最佳实践](https://developer.android.com/kotlin/coroutines/coroutines-best-practices)。

**面试时可以这样说：**

> 协程通过挂起和恢复组织异步任务，实际执行还依赖线程。suspend 只是允许挂起，不会自动切线程。代码在哪里运行要看调度器，耗时计算和阻塞调用都需要明确安排执行位置。

## 2. launch、async、withContext 有什么区别？

先看返回值和调用目的：

| API | 返回值 | 典型用途 |
|---|---|---|
| `launch` | `Job` | 启动任务，不取结果值 |
| `async` | `Deferred<T>` | 启动计算，再等待结果 |
| `withContext` | 结果 `T` | 指定上下文，等待完成 |

`launch` 和 `async` 都是 `CoroutineScope` 的扩展函数，默认会调度执行新协程，调用方拿到任务句柄后可以继续往下执行。它们不保证启动到另一条线程，也不保证任务一定在调用返回后才开始执行。`Deferred` 本身也是一个 `Job`。参见 [协程组合](https://kotlinlang.org/docs/composing-suspending-functions.html)。

`withContext` 是挂起函数，调用方要等它完成才能继续后面的逻辑。等待期间可以挂起，不需要阻塞调用线程。它会遵守结构化并发，等待块内的子协程完成。参见 [withContext API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-context.html)。

```kotlin
viewModelScope.launch {
    val result = withContext(Dispatchers.Default) {
        calculateSummary()
    }
    showResult(result)
}
```

这里用 `launch` 启动页面任务，用 `withContext` 安排计算，计算完成后再更新状态。

还有一个追问：**withContext 一定切换线程吗？**

不一定。上下文除了调度器，还能包含协程名称等元素。如果调度器不变，通常不需要额外调度；即使改变调度器，也不能把它理解成必定换一条物理线程，例如 `IO` 与 `Default` 会共享底层线程资源。参见 [Dispatchers.IO 的实现说明](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-i-o.html)。

**面试时可以这样说：**

> launch 返回 Job，适合启动不需要返回值的任务；async 返回 Deferred，适合并发计算后取结果；withContext 在指定上下文执行一段工作，等待完成并直接返回结果。它们都不等于创建新线程。

## 3. async 加 await，为什么有时仍然是串行？

看这段代码：

```kotlin
coroutineScope {
    val user = async { api.loadUser() }.await()
    val positions = async { api.loadPositions() }.await()
    render(user, positions)
}
```

第一个 `await()` 返回后，第二个任务才创建，因此这两个请求没有并发执行。

如果它们互不依赖，应当先启动，再等待：

```kotlin
coroutineScope {
    val userTask = async { api.loadUser() }
    val positionsTask = async { api.loadPositions() }

    render(userTask.await(), positionsTask.await())
}
```

**两个 await 顺序写，不代表两个任务串行运行。** 默认启动方式下，两个任务已经启动，`await()` 负责等待结果。

如果使用 `CoroutineStart.LAZY`，任务不会在创建时启动。想让多个懒启动任务并发运行，可以先分别调用 `start()`，再等待结果。单个任务创建后马上 `await()`，通常没有必要引入 `async`；直接调用挂起函数，或按需使用 `withContext` 即可。这些行为可以对照 [协程组合中的并发与懒启动示例](https://kotlinlang.org/docs/composing-suspending-functions.html)。

**面试时可以这样说：**

> 并发取决于任务何时启动。先 async 再立即 await，会等这个任务完成才创建下一个；先创建多个默认启动的 async，再依次 await，它们就可以并发推进。是否真正并行还取决于调度器和任务类型。

## 4. coroutineScope 和 supervisorScope 怎么选？

这两个函数都会等待自己的代码块和子协程结束，也都会随调用方取消。

区别主要在子任务失败后，其他任务是否还应该继续：

- `coroutineScope`：普通子协程发生非取消异常时，整个作用域失败，其他子协程也会被取消。
- `supervisorScope`：某个直接子协程失败，不会因此取消其他直接子协程。

例如，一个页面必须同时拿到用户信息和仓位才能展示，可以用 `coroutineScope`；行情模块和公告模块允许独立失败，可以考虑 `supervisorScope`。

但 `supervisorScope` 不会替你处理异常：

```kotlin
suspend fun loadOptionalNotice(): String? = supervisorScope {
    val task = async { api.loadNotice() }
    try {
        task.await()
    } catch (e: CancellationException) {
        throw e
    } catch (e: IOException) {
        null // 明确允许公告请求失败
    }
}
```

如果 `await()` 抛出的异常没有捕获，逃出了 `supervisorScope` 的代码块，那么代码块本身失败，仍然会取消该作用域里的其他任务。直接子 `launch` 的未捕获异常也需要处理，不能因为使用了监督作用域就忽略。参见 [supervisorScope API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/supervisor-scope.html)。

**面试时可以这样说：**

> 多个任务共同组成一个结果，用 coroutineScope，失败时一起取消。任务允许独立失败时用 supervisorScope，并分别处理异常。监督作用域隔离的是子任务失败的传播，不是自动吞掉异常。

![coroutineScope 与 supervisorScope 的异常传播对比](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/k1.png)

## 5. async 的异常只有 await 时才会抛吗？

这个问题要区分两件事：**获取失败结果**和**向父协程传播失败**。

`async` 会把成功结果或异常保存在 `Deferred` 中，`await()` 会返回结果或重新抛出异常。但在普通父子关系下，子 `async` 发生非取消异常时，也会使父协程失败，并取消其他子协程。这个过程不需要等待 `await()`。参见 [async API 的异常说明](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html)。

下面的代码没有调用 `await()`，外层仍然能收到异常：

```kotlin
suspend fun demonstrateFailure() {
    try {
        coroutineScope {
            async<Unit> {
                throw IOException("仓位请求失败")
            }
        }
    } catch (e: IOException) {
        println("请求失败：${e.message}")
    }
}
```

如果要把整组并发任务当作一次操作处理，可以像上面这样在 `coroutineScope` 外捕获。只在普通作用域内部捕获某个 `await()` 的异常，不能把已经失败的作用域恢复成正常状态。

`CoroutineExceptionHandler` 则负责最终未捕获异常的兜底处理，不能让失败的协程从出错位置继续。安装在 `async` 上的 handler 不会直接处理它保存在 `Deferred` 中的异常，但子 `async` 的失败可能沿普通父子关系传播，最终触发父级 `launch` 的异常处理。参见 [协程异常处理](https://kotlinlang.org/docs/exception-handling.html)。

**面试时可以这样说：**

> async 把异常保存在 Deferred 中，await 会重新抛出；但普通子 async 的失败也会传播给父协程，不调用 await 仍可能导致整个作用域失败。判断异常行为，必须同时看 builder 和父子关系。

## 6. 调用 cancel，协程就会立刻停止吗？

协程取消是协作式的。`cancel()` 发出取消请求，代码要通过可取消的挂起函数或主动检查来响应。

`cancel()` 也不会等待任务和清理逻辑结束；如果后续操作必须等旧任务退出，可以使用 `cancelAndJoin()`。它组合了取消和等待，但仍然依赖任务配合取消，不能强制终止不响应取消的代码。参见 [Job 的取消与等待说明](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html#communicating-with-the-coroutine)。

`delay()` 等函数支持取消；持续计算、没有取消检查的循环，可能继续占用线程。可以使用 `ensureActive()`，或者在需要让出执行机会时使用 `yield()`。参见 [ensureActive API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/ensure-active.html)。

```kotlin
withContext(Dispatchers.Default) {
    for (position in positions) {
        ensureActive()
        calculatePosition(position)
    }
}
```

检查发生在两次计算之间。如果单次 `calculatePosition()` 本身耗时很长，还要在它内部加入检查，或者把计算拆小。

异常处理也要保留取消语义：

```kotlin
try {
    repository.refresh()
} catch (e: CancellationException) {
    throw e
} catch (e: IOException) {
    showNetworkError(e)
}
```

当确实需要捕获宽泛的 `Exception` 时，应先重新抛出 `CancellationException`。普通 `runCatching` 也会捕获取消异常，不能不加区分地把失败统一转换成默认数据。Android 官方也提醒不要吞掉取消异常，参见 [协程最佳实践的取消与异常处理](https://developer.android.com/kotlin/coroutines/coroutines-best-practices)。

**面试时可以这样说：**

> cancel 发出的是取消请求。挂起函数和计算代码都需要配合，CPU 循环可以调用 ensureActive。捕获异常时要保留 CancellationException，否则页面已经离开，任务还可能继续执行后续逻辑。

## 7. by lazy 和 lateinit 有什么区别？

它们都能让属性晚一点初始化，但初始化方式不同。

```kotlin
val parser by lazy { PositionParser() }

lateinit var adapter: PositionAdapter
```

`parser` 在第一次读取时执行初始化函数，成功后缓存结果。`adapter` 则需要代码在合适的位置主动赋值。

| 对比项 | `by lazy` | `lateinit` |
|---|---|---|
| 声明要求 | 标准委托用于 `val` | 必须是 `var` |
| 初始化时机 | 第一次读取 | 主动赋值 |
| 初始化之后 | 返回缓存值 | 可以再次赋值 |
| 可空类型 | 支持 | 不支持 |
| `Int` 等基本类型 | 支持 | 不支持 |
| 提前访问 | 执行初始化函数 | 抛出未初始化异常 |

这里的未初始化异常是 `UninitializedPropertyAccessException`。

`lazy()` 返回 `Lazy<T>` 委托，读取属性时会读取委托的值。初始化抛异常时不会缓存一个成功结果，下次读取还会尝试初始化。参见 [属性委托](https://kotlinlang.org/docs/delegated-properties.html)与 [JVM lazy 实现](https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/jvm/src/kotlin/util/LazyJVM.kt)。

JVM 上的 `lateinit` 属性通常通过后备字段保存值，未初始化时字段为 `null`，getter 在读取时检查并抛出相应异常。它不自动加锁，也不自动提供跨线程可见性保证。

在允许访问该属性后备字段的位置，可以用 `::adapter.isInitialized` 判断是否初始化；这只是状态检查，不构成线程同步。参见 [延迟初始化属性](https://kotlinlang.org/docs/properties.html#late-initialized-properties-and-variables)。

**面试时可以这样说：**

> lazy 是第一次访问时自动计算并缓存，适合按需创建的只读属性；lateinit 是先声明、后赋值，适合初始化时机由外部流程决定的可变属性。lateinit 不支持可空类型和基本类型，访问前必须赋值，本身没有线程安全保证。

## 8. lazy 是线程安全的吗？底层怎么实现？

Kotlin/JVM 的 `lazy` 默认使用 `SYNCHRONIZED`，另外还有 `PUBLICATION` 和 `NONE`。三个模式的保证见 [LazyThreadSafetyMode API](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin/-lazy-thread-safety-mode/)。

**SYNCHRONIZED：通过锁协调初始化。**

JVM 实现使用 `volatile` 值字段和同步锁：先检查有没有缓存值，没有再加锁；进入锁后再次检查，确实未初始化才调用初始化函数。成功后缓存结果，后续读取可以直接返回。这是双重检查的思路。初始化失败后仍可能重试，所以“初始化函数永远只执行一次”并不严谨。

**PUBLICATION：允许重复计算，只发布一个结果。**

竞争时多个线程可能执行初始化函数，JVM 实现通过 CAS 竞争发布结果，最后所有调用方使用同一个已发布值。初始化函数如果会注册监听、创建连接或产生其他副作用，就要谨慎选择这个模式。

**NONE：不提供线程安全保证。**

当能保证属性始终由同一线程访问时，可以考虑使用它。具体实现可查 [LazyJVM.kt](https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/jvm/src/kotlin/util/LazyJVM.kt)。

还要分清初始化安全与对象使用安全：

```kotlin
val positions by lazy { mutableListOf<String>() }
```

默认模式能协调这个列表的初始化，但多个线程同时对列表执行 `add()`、`remove()`，仍然需要同步措施。`val` 也只限制引用重新赋值，不会把集合变成不可变集合。

**面试时可以这样说：**

> lazy 默认的 SYNCHRONIZED 保证初始化和结果发布的线程安全，JVM 上通过 volatile 和同步锁实现；PUBLICATION 允许多个线程计算，再通过 CAS 发布一个结果；NONE 没有线程安全保证。这些保证都不会自动覆盖对象后续的可变操作。

## 9. Flow 是什么？冷流和热流有什么区别？

挂起函数常用来返回一个结果；Flow 用来表示异步数据序列，例如行情、数据库变化和页面状态变化。

用 `flow {}` 创建的流是冷流：有人收集时才执行流里的代码，每次收集都会重新执行。

```kotlin
val priceFlow = flow {
    println("开始请求")
    emit(api.loadPrice())
}

priceFlow.collect { println(it) }
priceFlow.collect { println(it) }
```

假设请求正常完成，这里会请求两次。因此，多个地方收集同一个冷流，可能造成重复请求或重复注册监听。

`StateFlow` 和 `SharedFlow` 是热流，多个订阅者共享同一个实例的状态或广播数据。`Flow` 是接口，不能说“所有 Flow 都是冷流”。`stateIn` 和 `shareIn` 可以在指定作用域中共享上游。参见 [Flow API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-flow/)。

另一个常见追问是 `flowOn`：它影响前面的上游操作，不会把后面的 `collect` 一起切到指定调度器。

```kotlin
viewModelScope.launch {
    flow { emit(readBlockingFile()) }
        .flowOn(Dispatchers.IO)
        .collect { content ->
            updateUiState(content)
        }
}
```

这里阻塞读取在 `IO` 执行，收集代码保留调用方上下文。参见 [flowOn API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow-on.html)。

**面试时可以这样说：**

> Flow 表示异步数据流。flow builder 创建的是冷流，每次收集都会重新执行；StateFlow 和 SharedFlow 是热流，多个订阅者共享数据。实际使用要关注是否重复启动上游，以及上游和收集代码分别在哪个上下文执行。

## 10. StateFlow 和 SharedFlow 有什么区别？

`StateFlow` 适合回答“当前是什么状态”；`SharedFlow` 提供更通用的广播能力。

| 对比项 | `StateFlow` | `SharedFlow` |
|---|---|---|
| 初始值 | 必须有 | 不要求 |
| 读取当前值 | 支持 `value` | 没有当前值属性 |
| 新订阅者 | 收到当前最新状态 | 根据 `replay` 收到历史数据 |
| 相同值 | 按 `equals` 合并 | 不自动按相等性去重 |
| 缓冲方式 | 保留最新状态 | 可配置回放和缓冲 |

StateFlow 的慢订阅者可能跳过中间状态；SharedFlow 可以配置回放数量、额外缓冲容量和溢出策略。

![StateFlow 与 SharedFlow 的重复值处理和新订阅者行为对比](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/k2.png)

`StateFlow` 是 `SharedFlow` 的一种特化，但面试中只说“等价于 replay 为 1”不够，还要补上初始值、相等性合并和当前值访问。参见 [StateFlow API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/)。

页面状态可以这样维护：

```kotlin
data class PositionsUiState(
    val loading: Boolean = false,
    val count: Int = 0
)

private val _uiState = MutableStateFlow(PositionsUiState())
val uiState: StateFlow<PositionsUiState> =
    _uiState.asStateFlow()
```

对 `SharedFlow`，几个参数需要分别理解：

- `replay`：新订阅者可以收到多少条历史值。
- `extraBufferCapacity`：为落后的现有订阅者提供额外缓冲。
- `onBufferOverflow`：缓冲满时选择挂起发送者、丢最旧值或丢新值。

**没有订阅者时，只有 replay 对应的历史值会保留。** 设置额外缓冲，并不意味着无人订阅时也能把事件攒起来。`replay = 0` 时，无订阅者期间发送的事件会丢失。参见 [SharedFlow API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-shared-flow/)。

**面试时可以这样说：**

> StateFlow 表示当前状态，必须有初始值，按 equals 合并更新，新订阅者拿最新值。SharedFlow 是可配置的广播流，可以回放历史，也允许发送重复值。使用它传事件时，要先明确无人订阅期间是否允许丢失。

## 11. StateFlow 线程安全，为什么 value++ 还可能丢数据？

`MutableStateFlow` 的读写方法是线程安全的，但“读取旧值、计算新值、写回”组合起来不自动具备原子性。

```kotlin
counter.value = counter.value + 1
```

两个线程可能同时读到 10，然后都写入 11，丢失一次更新。

应该使用原子更新：

```kotlin
counter.update { old -> old + 1 }

_uiState.update { old ->
    old.copy(count = old.count + 1)
}
```

`update` 的函数可能在竞争时执行多次，因此里面适合计算新状态，不适合执行下单、发请求等副作用。参见 [update API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/update.html)。

还要避免直接修改状态内部的可变集合，再把同一个对象赋回去。`StateFlow` 按相等性合并更新，这种写法可能没有新的通知，其他读取者也可能看到你修改到一半的数据。实际项目通常使用不可变状态快照，并创建新的集合和状态对象。相等性规则见 [StateFlow 文档](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/#strong-equality-based-conflation)。

**面试时可以这样说：**

> 容器的单次操作线程安全，不代表多步读改写是原子的。根据旧值更新 StateFlow 时用 update，里面保持无副作用；状态对象本身也要避免共享可变数据。

## 12. 页面退出后，为什么 Flow 还在刷新？

先确认“退出”指什么：页面不可见、Fragment 的 View 销毁、ViewModel 清除，是不同的时机。

普通 `lifecycleScope.launch { collect(...) }` 不会因为页面收到 `ON_STOP` 就自动取消收集。这里要区分事件和状态：AndroidX `Lifecycle.State` 没有 `STOPPED`，停止后的状态通常是 `CREATED`。Fragment 的 UI 收集可以放在 `onViewCreated()` 中，通过 View 的生命周期管理：

```kotlin
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state ->
            render(state)
        }
    }
}
```

生命周期低于 `STARTED` 时，内部收集协程会被取消；重新达到 `STARTED` 时会启动新的收集。这里依据的是 Lifecycle 状态，不能把任意页面遮挡、Tab 切换都视为已经触发停止；如果这些操作没有改变生命周期，还需要显式管理订阅。参见 [Android 生命周期与协程指南](https://developer.android.com/topic/libraries/architecture/coroutines)。

**停止收集，还不等于停止数据生产。** 如果 Service 仍在接收行情和执行计算，页面这边取消 `collect` 不会自动停止那些独立任务。

如果希望没有订阅者时停止当前这条流的上游收集，可以在 ViewModel 中共享数据：

```kotlin
val uiState = repository.observePositions()
    .map { positions ->
        PositionsUiState(count = positions.size)
    }
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(
            stopTimeoutMillis = 5_000
        ),
        initialValue = PositionsUiState(loading = true)
    )
```

最后一个订阅者离开 5 秒后，这里的共享协程会停止收集上游。这个时间用来容忍短暂的订阅空档，可以按业务调整；它不会在默认配置下清除缓存。参见 [WhileSubscribed API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-sharing-started/-companion/-while-subscribed.html)。

上面的示例只展示订阅管理。实际接入网络数据时，还应在 `stateIn` **之前**用 `catch` 将可预期异常转换为错误状态，或通过有次数、间隔限制的 `retry` / `retryWhen` 重试。未经处理的上游异常会终止共享协程，由传入的 scope 处理，并不会作为流中的异常直接传给 StateFlow 订阅者；`WhileSubscribed` 也不能代替异常重试。参见 [stateIn 的上游异常处理](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/state-in.html#upstream-completion-and-error-handling)。

上游仍需配合释放资源。如果用 `callbackFlow` 包装监听，应在 `awaitClose` 中移除对应监听或取消对应订阅；若底层 WebSocket 由多个页面共用，就应管理各业务订阅，不能直接关闭整条连接。参见 [callbackFlow API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/callback-flow.html)。

**面试时可以这样说：**

> UI 收集用 repeatOnLifecycle，在生命周期低于 STARTED 时停止处理。生产端则要单独管理，可以结合 WhileSubscribed 和上游取消清理。共享连接、业务订阅、页面渲染是不同层级，需要分别决定生命周期。

## 13. 高频行情处理不过来，buffer、conflate、collectLatest 怎么选？

先判断业务是否允许跳过数据，再选操作符。

**每一条都需要处理时，可以使用有界 buffer。** 默认溢出策略会在缓冲满时挂起上游。缓冲能吸收短时峰值，但不能解决长期“生产快、消费慢”的问题，也不能自动让外部 WebSocket 服务器减速。参见 [buffer API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/buffer.html)。

**只关心最新待处理值时，可以使用 conflate。** 它允许跳过积压的中间值，但不会取消当前正在处理的值。参见 [conflate API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/conflate.html)。

**新值到来后，旧处理已经没有意义时，可以使用 collectLatest。** 它取消前一次 action，再处理新值；action 内部也必须响应取消。参见 [collectLatest API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/collect-latest.html)。

下面的收集代码应放在主线程、绑定页面生命周期的协程中，例如上一题的 `repeatOnLifecycle` 代码块内，让 `render()` 在主线程执行：

```kotlin
priceSnapshots.collectLatest { snapshot ->
    val rows = withContext(Dispatchers.Default) {
        snapshot.positions.map { position ->
            ensureActive()
            calculateRow(position, snapshot.prices)
        }
    }
    render(rows)
}
```

这里假设每个 snapshot 都是足够计算当前展示的完整快照。若输入一直比计算快，频繁取消也可能导致迟迟没有结果，此时要考虑固定展示节奏、减少单次计算量或按需更新。

需要限制展示频率时，可以考虑 `sample`，在采样窗口内取最新值；但它可能不发出有限流末尾尚未到采样时机的值，使用时也要留意 API 的 `FlowPreview` 标记。参见 [sample API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/sample.html)。

仓位场景还要注意：完整价格快照通常允许跳过中间展示；“数量加 1”“扣减一笔”这类增量事件不能随意丢弃。应先按协议还原可信状态，再对展示层的快照合并或采样。

**面试时可以这样说：**

> buffer 处理短时积压，conflate 保留最新待处理值，collectLatest 会取消旧处理。选择依据是数据能不能丢、旧处理能不能取消。增量业务事件与 UI 快照要分开，不能为了流畅把状态算错。

## 14. 1,500 个仓位，每个都开一个协程，就没有性能问题了吗？

协程比平台线程轻量，但它不会减少计算量。每条行情都为 1,500 个仓位新建任务，仍然可能出现调度开销、任务积压、对象分配、旧结果覆盖新结果，以及主线程刷新过密的问题。

面对这道题，可以按任务类型回答：

**如果只是短小计算，先评估批量执行。** 不必为几次乘加运算创建一个协程。可以在 `Default` 上批量处理，按行情品种复用公共计算，并定期检查取消。

**如果是较重的 CPU 计算，再考虑有界并行。** 可以复用一个 `Dispatchers.Default.limitedParallelism(n)` 调度器视图控制并行执行量。`n` 要结合耗时和设备能力测量，不能直接设成仓位数量。

这里的限制只约束同时执行代码的并行度，**不会限制已经挂起、尚未完成的网络请求数量**。`limitedParallelism(1)` 也不等于用锁保护一整段包含挂起点的业务操作。参见 [limitedParallelism API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-dispatcher/limited-parallelism.html)。

**如果要限制并发请求数，可以使用 Semaphore。**

```kotlin
// 作为 Repository 的复用字段，8 仅为示例值
private val requestSlots = Semaphore(permits = 8)

suspend fun loadPosition(id: String): Position =
    requestSlots.withPermit {
        api.loadPosition(id)
    }
```

许可会覆盖请求的整个执行过程，包括挂起等待。参见 [Semaphore API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-semaphore/)。

Semaphore 不会减少已经创建、正在等待许可的协程数量。如果输入源源不断，还应使用有界任务队列、固定数量的处理协程或批次处理，限制待处理任务总量。

**面试时可以这样说：**

> 把线程换成协程，只降低了一部分调度和资源成本。短计算优先批量处理，重计算控制并行度，请求通过 Semaphore 限制在途数量，还要控制队列长度、取消过期任务和限制 UI 更新频率。

## 15. 以前用 EventBus 推送数据，换成 Flow 怎么设计？

迁移时先把消息的用途分清楚，再选择数据流类型。

仓位列表、连接状态、当前行情属于持续状态，可以由 Repository 接收推送并维护，再向 ViewModel 暴露只读 Flow。ViewModel 转换成页面 `UiState`，页面按生命周期收集。

临时广播可以用 `SharedFlow`，但要回答清楚：无人订阅时是否允许丢失？页面重建后要不要重放？多个订阅者是否都要处理？

例如 Toast 提示是否允许错过，可以由产品体验决定；平仓是否成功，则必须能够从可信状态或服务端查询结果中恢复，不能只依赖一条可能错过的页面事件。

`SharedFlow` 的 `replay = 1` 也不代表“只消费一次”。新订阅者会再次收到缓存值；如果要实现可靠的业务处理，需要额外设计事件标识、确认、持久化或幂等机制。广播与回放语义见 [SharedFlow 官方说明](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-shared-flow/)。

一种可行的数据路径是：

```text
WebSocket 回调
    ↓
Repository：校验顺序、合并增量、维护业务状态
    ↓
ViewModel：组合页面状态、安排展示计算
    ↓
StateFlow<UiState>
    ↓
页面按生命周期收集并渲染
```

这里的关键收益是依赖关系和数据所有者变得明确。Flow 本身不负责修复推送乱序、不保证业务消息不丢，也不自动完成生命周期管理，这些仍然需要在各层实现。

**面试时可以这样说：**

> 我会先把持续状态和临时广播分开。业务状态由 Repository 维护，ViewModel 暴露 StateFlow 给页面；允许广播的事件按丢失和重放要求配置 SharedFlow。迁移时同时梳理订阅生命周期、背压和一致性，避免只把全局 EventBus 换成一个全局 SharedFlow。

---

复习时可以拿“仓位页接收行情、计算盈亏、更新列表”这一条业务链路练习：任务由谁启动，在哪里计算，失败影响谁，页面离开后谁负责取消，新旧数据怎样衔接。能把这些选择讲清楚，再遇到换了问法的协程题，就有具体代码和业务条件可以展开。

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**
