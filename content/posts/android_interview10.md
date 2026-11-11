---
title: "Android 面试高频十问：从组件、协程到 Compose"
draft: false
categories: ["Android"]
tags: ["Android", "Kotlin", "协程", "Jetpack Compose", "面试"]
description: "梳理 Android 面试中的十个常见问题，讲清协程调度、Flow 背压、Compose 状态恢复与重组，以及状态和事件的传递边界。"
cover:
  image: "https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/ar10_0.png"
  alt: "Android 面试十问：组件、协程与 Compose"
  hiddenInSingle: true
---

面试官问“`StateFlow` 和 `SharedFlow` 怎么选”，你可能说得出定义，却还需要回答：页面重建后，事件该不该重放？没有收集者时，事件去了哪里？

这类题会追问你对状态、并发和生命周期的判断。本文整理十个常见问题，结合代码说明原理、使用场景和边界条件，适合面试前复习，也方便对照项目中的实现。

> 阅读建议：别急着往下翻答案。先用自己的话回答每个标题，再核对遗漏的边界条件。

![Android 面试十问封面：手机界面、数据访问与组件结构示意](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/ar10_0.png)

## 1. 什么是 ContentProvider？什么时候使用它？

`ContentProvider`（内容提供器）是 Android 四大组件之一，另外三个是 `Activity`、`Service` 和 `BroadcastReceiver`。它用于以统一、受控的方式向其他应用暴露数据，是跨进程通信（IPC）的常见方案。

可以把它理解为“可对外共享的数据访问层”：应用将本地数据库、文件或其他数据源包装为内容提供器；其他应用通过 `ContentResolver` 和 URI 查询、插入、更新或删除数据。提供方可以配合权限精确限制访问范围。

适合使用它的场景是：数据不只服务于当前应用，还需要被其他应用安全访问。例如：

- 通讯录：系统通过 `ContactsContract` 暴露联系人数据；
- 媒体库：`MediaStore` 提供系统管理的图片、音频、视频等媒体元数据；
- 自己的应用需要向可信的第三方应用提供结构化数据。

若数据只在应用内部使用，通常直接访问 Room、文件或仓库层即可，不必为了“规范”而引入 `ContentProvider`。

## 2. 协程和线程有什么区别？

线程是操作系统调度的执行单元，创建和切换都有成本。每个 Android 进程至少有一个主线程；多个线程可以利用多核 CPU 真正并行，也可以在单核上通过调度交替执行。

协程是运行在线程之上的轻量级并发抽象。调度器决定协程使用哪类线程；多个协程可以复用少量线程。它的关键能力是**挂起**：协程可以暂停执行，让出线程，稍后再恢复。`suspend` 允许函数挂起，但函数调用不一定实际挂起，也不会自动切换线程。

```kotlin
delay(1_000)          // 挂起协程，不阻塞线程
Thread.sleep(1_000)   // 阻塞当前线程
```

若在 `Dispatchers.Main` 中使用 `Thread.sleep()` 或其他阻塞调用，主线程无法绘制 UI，界面就会卡顿。协程并不会自动让阻塞代码变成非阻塞代码：阻塞 I/O 仍应切到 `Dispatchers.IO`。

## 3. `Dispatchers.Main`、`IO` 和 `Default` 有什么区别？

调度器决定协程在哪类线程上执行。协程本身不是线程；它最终仍需要运行在线程之上。

| 调度器 | 适用工作 | 为什么 |
| --- | --- | --- |
| `Dispatchers.Main` | 更新 UI、调用必须在主线程执行的 API | 运行在 Android 主线程；耗时或阻塞操作会造成掉帧甚至 ANR |
| `Dispatchers.IO` | 同步网络调用、文件读写、阻塞数据库访问等 | 执行阻塞 I/O，并在并行度限制内按需调度线程 |
| `Dispatchers.Default` | 图像处理、排序、大集合计算、压缩等 CPU 密集型工作 | 默认并行度针对可用 CPU 资源设计，避免过多线程争抢 CPU |

执行阻塞 I/O 的线程会等待结果。`IO` 允许更多这类任务并发执行，减少等待对其他任务的影响。CPU 密集型工作应交给 `Default`：CPU 接近满载时，继续增加线程往往只会增加上下文切换成本。`IO` 和 `Default` 在 JVM 上共享底层线程资源，切换调度器也不一定意味着换一条线程。

是否需要切到 `IO`，取决于调用是否阻塞当前线程。已经能安全从主线程调用的挂起 API，不需要调用方再统一包裹 `withContext(IO)`。下面假设 `fetchUserBlocking()` 是阻塞接口，演示一次调度器切换：

```kotlin
viewModelScope.launch {
    val user = withContext(Dispatchers.IO) { repository.fetchUserBlocking() }
    // 回到 Main，安全更新 UI 状态
    uiState.value = UiState.Success(user)
}
```

实际项目中，建议由 Repository 内部切换执行阻塞操作所需的调度器，对外提供可安全从主线程调用的挂起函数，让调用方只关心业务流程。

## 4. Kotlin Flow 中的背压（Backpressure）是什么？

生产速度超过消费能力时，需要协调两端的速度。背压通常指下游通过等待等机制约束上游，避免生产者持续发送来不及处理的数据。

普通 `flow { ... }.collect { ... }` 默认顺序执行：下游没有处理完当前值，上游的 `emit` 就不会返回。例如，上游每次先 `delay(300)` 再发射，下游处理一项需要 500 ms，那么相邻两次发射的间隔约为 800 ms，不会自动堆出一个不断增长的队列。

引入缓冲、并发边界或外部事件源后，就需要明确如何处理生产与消费的速度差：

- `buffer()`：让生产与消费在不同协程中并发推进，通过队列暂存数据。默认使用有限缓冲，缓冲满时挂起生产者，适合需要按序处理数据的场景。缓冲只能吸收短期速度差，配置无界或过大的缓冲会带来内存压力。
- `conflate()`：消费者忙碌时只保留最新的待处理值，跳过旧的待处理值，但会继续完成当前处理。适合只关心最新 UI 状态、传感器读数等场景。
- `collectLatest { }`：新值到达时取消仍在执行的上一轮处理，再处理新值。适合搜索联想等旧结果已经失去意义的场景，前提是处理逻辑能够配合取消。

```kotlin
searchQuery
    .debounce(300)
    .collectLatest { query ->
        // 防抖后的新查询到达时，取消上一轮处理
        searchRepository.search(query)
    }
```

这里先经过 `debounce(300)`，因此新输入要等防抖结束、传到 `collectLatest` 后，才会触发取消；等待期间旧搜索仍可能完成。底层网络请求能否一起取消，取决于所用 API 的实现。`debounce` 标注为 `FlowPreview`，可在所在函数或类上添加 `@OptIn(FlowPreview::class)` 显式选择使用。

选择策略前先问：每一条事件都必须处理吗？能否丢弃旧值？正在进行的工作能否安全取消？

![Flow 默认顺序执行、buffer、conflate 与 collectLatest 的处理过程对比](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/ar10_1.png)

## 5. `StateFlow`、`SharedFlow` 与 `Channel` 如何选择？

三者都常用于协程中的数据传递，但建模目标不同。

| 工具 | 最适合表达 | 订阅与重放特征 |
| --- | --- | --- |
| `StateFlow` | 当前状态 | 始终有当前值；新订阅者获得最新值；相等值不重复通知，慢收集者可能跳过中间更新 |
| `SharedFlow` | 可广播的事件流 | 同一次发射可分发给多个收集者；能否收到历史值或跟上更新，取决于 `replay`、缓冲和溢出策略 |
| `Channel` | 队列式消息传递 | 可有多个接收者竞争接收，但同一元素只交给其中一个；可通过 `receiveAsFlow()` 暴露 |

```kotlin
private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
val uiState = _uiState.asStateFlow()

private val _events = MutableSharedFlow<UiEvent>()
val events = _events.asSharedFlow()
```

页面要渲染“现在是什么状态”，优先用 `StateFlow`。它按 `equals` 合并相等值，慢收集者可能跳过快速更新中的中间状态，因此不适合记录每一次事件。

一个事件需要通知多个观察者时，可考虑 `SharedFlow`。默认 `MutableSharedFlow()` 的 `replay` 和额外缓冲容量都为 0：有订阅者时，`emit()` 会等待订阅者接收；没有订阅者时，它会立即返回，事件不会保留。接收也不代表订阅者已完成业务处理。

没有订阅者时，`SharedFlow` 只保留 `replay` 指定数量的历史值，`extraBufferCapacity` 不会替未来订阅者缓存事件。默认配置下，此时 `tryEmit()` 即使返回 `true`，也不代表有人收到。设置 `replay` 能让新订阅者收到历史值，但也要考虑页面重建后重复执行 UI 效果的问题。

`Channel` 适合点对点通信，也适合多个工作协程分摊任务。每个元素只交给其中一个接收者，但接收不保证业务处理成功：例如，`receiveAsFlow()` 的收集协程可能在接收后、处理前取消。它也不保证固定轮询分配任务。

对于 Snackbar、导航等一次性 UI 效果，不应只因“它是事件”就机械选择某一种工具；还要定义清楚页面未订阅、旋转重建或重复收集时，事件究竟应该丢弃、保留还是重放。

这三者本身都不提供跨进程重启的持久化保证。需要可靠恢复的业务结果，应结合可恢复状态或持久化存储设计。

![StateFlow 保存当前状态、SharedFlow 广播事件与 Channel 多接收者竞争接收的区别](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/ar10_3.png)

## 6. `remember` 与 `rememberSaveable` 的区别

两者都可以在 Compose 中保留跨重组（recomposition）的状态。`remember` 会在对应组合位置仍然存在、key 没有变化时复用之前保存的值。

```kotlin
var count by remember { mutableIntStateOf(0) }
var selectedTab by rememberSaveable { mutableIntStateOf(0) }
```

`rememberSaveable` 还会通过保存实例状态机制，将可保存的值写入 `Bundle`。在系统已保存并提供恢复数据的前提下，它可以恢复配置变更（如旋转屏幕）或系统回收进程后重建页面时的状态。它不等于持久化存储，用户结束页面或移除任务后重新打开，不能依赖它恢复之前的内容。

`rememberSaveable` 适合 `String`、数字等 Bundle 可保存类型，或可通过 `Saver` 转换的轻量数据。不要把大型对象、位图或复杂业务状态塞进去。需要业务逻辑管理的页面状态，更适合放在 `ViewModel`；跨屏幕共享时，还要选择合适的 ViewModel 作用域。

`ViewModel` 可以跨配置变更保留状态，但不会跨进程死亡存活。需要在进程重建后恢复时，可以用 `SavedStateHandle` 保存查询条件、条目 ID 等少量信息，再从数据层重新加载完整内容。状态由谁持有、重建后如何恢复，是两个需要分别决定的问题。

## 7. Compose 如何知道哪些内容需要重组？

Compose 通过 Snapshot 状态系统记录状态读取与作用域的依赖关系。在组合阶段读取 `State<T>.value` 时，Compose 会记录这次读取；状态发生有效变化后，对应的可重组作用域会失效，等待重新执行。

```kotlin
@Composable
fun CounterSection() {
    val count = remember { mutableIntStateOf(0) }

    Column {
        CountText(count.intValue) // 在组合阶段读取 Snapshot 状态
        IncreaseButton(onClick = { count.intValue++ })
    }
}

@Composable
fun CountText(count: Int) {
    Text("当前计数：$count")
}

@Composable
fun IncreaseButton(onClick: () -> Unit) {
    Button(onClick = onClick) {
        Text("增加")
    }
}
```

这里的状态读取发生在 `CounterSection` 调用 `CountText`、计算实参的位置。点击按钮修改状态后，`CounterSection` 的重组作用域失效；重新执行时，`CountText` 收到新的 `Int` 参数并更新文字。`CountText` 内部读取普通 `Int` 参数，并不等于读取 Snapshot 状态。

其他 Composable 是否重新执行，还取决于作用域边界、参数变化和编译器生成的跳过逻辑。也不是每次状态变化都会触发重组：如果只在布局或绘制阶段读取状态，Compose 可以只重新执行受影响的阶段。

实践上，状态应尽量向下传递、事件向上传递，并避免在组合过程中读取不必要的可变状态。这样依赖边界清晰，也能缩小重组范围。

![Compose 在 CounterSection 中读取状态，并在状态变化后使对应重组作用域失效](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/ar10_2.png)

*图示前提：Header、CounterSection、Footer 是 Screen 下的同级组件，图中纵向排列表示页面位置；只有 CounterSection 在组合阶段读取该状态。绿色区域表示本次受影响的重组范围，其中参数变化的子组件也可能重新执行。*

## 8. `val` 和 `const val` 有什么区别？

主要区别是值能否在编译期确定，以及声明位置和类型的限制：

- `val` 是只读变量，值可在运行时计算；可以声明为局部变量、成员属性等。
- `const val` 是编译期常量，值必须在编译时就能确定，只能声明为顶层或 `object` / `companion object` 中的属性，类型限于基本类型和 `String`，且不能有自定义 getter。

```kotlin
const val MAX_RETRY = 3
val greeting = "hello".reversed() // 运行时计算，不能是 const val
```

`val` 的只读限制不代表对象不可变。例如，`val items = mutableListOf(1)` 仍然允许 `items.add(2)`。声明为 `val` 的属性也可以有自定义 getter，每次读取时返回不同结果。

编译器会将 `const val` 的值内联到引用处，因此它常用于常量、注解参数等。也要注意：若常量来自独立库，修改并发布库后，使用方通常需要重新编译才能获得新值。

## 9. 什么是封装（Encapsulation）？

封装是面向对象设计的核心原则：明确对外公开的能力，并隐藏不需要被外部了解的实现细节。外部代码只依赖稳定的接口或行为，不依赖内部如何存储、计算或协作。

以基于数组实现的 `ArrayList` 为例，调用方需要知道 `add()`、`remove()`、索引访问等公开 API，但不需要了解底层数组如何扩容、移动或存储元素。`MutableList` 是接口，本身不限定底层存储结构。

在 Kotlin 中，可以利用可见性修饰符实现封装：

```kotlin
class Counter {
    var value: Int = 0
        private set

    fun increment() {
        value++
    }
}
```

这里外部可以读取 `value`，却不能任意修改它，只能通过 `increment()` 改变状态。这样的边界能保护不变量，降低模块耦合，并让内部实现可以安全演进。

## 10. Kotlin 中的 `Any`、`Unit` 与 `Nothing`

这三个类型分别表达完全不同的含义：

| 类型 | 含义 | 常见用途 |
| --- | --- | --- |
| `Any` | Kotlin 非空类型的根类型，类似 Java 的 `Object` | 接受任意非空对象的参数，或显式非空泛型上界 `<T : Any>` |
| `Unit` | 只有一个值 `Unit` 的类型，表示没有有意义的返回值 | 不返回业务结果的函数，作用类似 Java 的 `void` |
| `Nothing` | 没有任何实例的底类型，函数不会正常返回 | 永远抛异常、无限循环、终止流程的函数 |

```kotlin
fun log(message: String): Unit = println(message)

fun fail(message: String): Nothing {
    throw IllegalStateException(message)
}
```

`Nothing` 的价值在于帮助编译器理解控制流。调用 `fail()` 后的代码不可达；它也能让 `?:`、`when` 等表达式保持正确的类型推导。

`Any` 不包含 `null`。若参数还要接受空值，应使用 `Any?`；泛型参数没有显式声明上界时，默认上界也是 `Any?`。

## 写在最后

把十道题全背下来，面试时也未必能答好。更实用的准备方式，是拿你做过的功能去验证这些概念：

- 列出项目中一个网络请求链路，说明哪里可能阻塞、由哪一层负责线程切换，以及结果如何更新 UI 状态；
- 找一个页面状态，先判断它应该留在 UI 层还是交给 `ViewModel`，再决定是否需要通过 `rememberSaveable` 或 `SavedStateHandle` 支持恢复；
- 找一个一次性事件，写清楚页面重建、没有收集者或多收集者时的预期行为。

能把代码里的选择说完整，比给出一段标准定义更有说服力。

如果这篇文章对你有帮助，欢迎转给正在准备 Android 面试的朋友。也欢迎在留言区补充你遇到过的高频题和项目里的真实取舍。

## 参考资料

- [Android 协程最佳实践](https://developer.android.com/kotlin/coroutines/coroutines-best-practices)
- [Flow：buffer](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/buffer.html)、[conflate](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/conflate.html)、[debounce](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/debounce.html)
- [StateFlow](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/)、[SharedFlow](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-shared-flow/)、[receiveAsFlow](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/receive-as-flow.html)
- [Compose 状态保存](https://developer.android.com/develop/ui/compose/state-saving)与[执行阶段](https://developer.android.com/develop/ui/compose/phases)
- [Kotlin 属性](https://kotlinlang.org/docs/properties.html)与[泛型约束](https://kotlinlang.org/docs/generics.html)

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**
