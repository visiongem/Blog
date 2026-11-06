---
title: "Android 事件通信与架构面试 12 问：从 EventBus、Flow 到 MVVM 与 MVI"
date: 2026-11-04T09:30:00+08:00
draft: false
isCJKLanguage: true
description: "整理 EventBus 迁移 Flow、StateFlow 与 SharedFlow 选型、Channel 事件投递、MVVM 与 MVI、数据倒灌及页面刷新，附生命周期收集和刷新状态示例。"
categories: ["Android"]
tags: ["Android", "Flow", "EventBus", "MVVM", "MVI", "面试"]
---

本篇整理 12 道 Android 事件通信与架构面试题，涵盖 EventBus 与 Flow 的选型、StateFlow 与 SharedFlow 的区别、生命周期收集，以及 MVVM、MVI、数据倒灌和页面刷新。

文章结合仓位推送、搜索和订单结果等场景，说明事件如何传递、状态由谁维护，以及页面重建和异步结果乱序时该如何处理。每道题都附有回答思路与面试总结。

示例使用 Kotlin、AndroidX Lifecycle 与 kotlinx.coroutines，省略常规 `import`；自定义业务类型和接口的约定会在代码旁说明。

![Android 事件通信与架构面试 12 问封面](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/kr0.png)

## 1. 数据推送为什么用 EventBus？什么时候值得换成 Flow？

EventBus 通过事件类型连接多个模块，发送方无需直接持有接收方。对于已有项目，EventBus 可能已经是团队长期采用的通信方式，是否迁移应结合维护成本和业务需求判断。

全局总线的依赖藏在订阅关系里：排查一个事件的处理过程，需要找到发送者、订阅者、线程模式，以及注册和注销的位置。业务越来越复杂时，多个订阅者还可能各自保存、修改一份仓位数据。

线程模式也不能忽略。EventBus 默认 `POSTING` 在发送线程同步调用；`MAIN` 在主线程发布时可直接调用订阅者，`MAIN_ORDERED` 则会排队分发。它们会影响事件是否排队，以及处理一个事件时再次发送事件的执行顺序。参见 [EventBus ThreadMode 文档](https://greenrobot.org/eventbus/documentation/delivery-threads-threadmode/)。

迁移到 Flow 的价值，通常来自明确的数据入口和组合关系：

```text
Socket / HTTP
→ Repository 归并业务数据
→ ViewModel 生成页面状态
→ UI 收集并渲染
```

Repository 暴露只读流，调用者通过持有的 Repository 获取数据，更容易追踪数据来源。Flow 操作符负责过滤、组合和错误处理，收集端再结合页面生命周期管理订阅。

如果只是把全局 EventBus 换成全局 `MutableSharedFlow<Any>`，所有模块仍然随意发消息、随意改状态，核心问题不会因为类型名字变化而消失。

**面试时可以这样说：**

> EventBus 适合已有的发布订阅通信，但要管好订阅生命周期和线程模式。迁移 Flow 时，我会先把业务状态集中到 Repository，再向页面暴露类型明确的流，不只是替换一个全局总线。

## 2. StateFlow、SharedFlow 和 Channel 怎么选？

先问：消费者需要“现在是什么”，还是“刚才发生过什么”？

| 工具 | 主要用途 |
| --- | --- |
| StateFlow | 当前状态 |
| SharedFlow | 广播通知 |
| Channel | 元素传递，多消费者分摊 |

StateFlow 必须有初始值，新订阅者获得当前值；它按 `equals()` 抑制相等更新，慢消费者可能跳过中间状态。页面只需展示当前仓位时，这通常符合需求。它不适合记录必须逐笔保留的成交历史。参见 [StateFlow API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/)。

SharedFlow 可以配置 `replay`、额外缓冲和溢出策略，适合广播，但是否能收到每次事件取决于订阅时机、缓冲配置与消费者速度。它不是持久化消息系统。

Channel 转成 `receiveAsFlow()` 后，多个消费者会分摊元素，一个元素只交给其中一个收集者，不会广播给所有页面。它也不必然带有缓冲：默认 `Channel()` 使用会合模式，即发送方与接收方配对后才能完成传递；没有接收者时，`send()` 会挂起等待。是否排队保存、能保存多少，要看容量配置。参见 [receiveAsFlow API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/receive-as-flow.html) 与 [Channel 容量说明](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel/)。

这些工具都不能独自保证进程重启后的恢复。进程重建后仍需恢复的业务状态或待处理任务，应持久化到合适的数据层。

**面试时可以这样说：**

> 当前状态用 StateFlow，需要广播时考虑 SharedFlow，任务分发可以考虑 Channel。选择前要讲清重放、缓冲、订阅时机和恢复要求，不能把 Channel 当广播，也不能把 Flow 当持久化队列。

![StateFlow 保存当前状态，SharedFlow 广播事件，Channel 让多个消费者分摊元素](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/kr1.png)

## 3. Socket 回调怎么接入 Flow？每个页面都会重复订阅吗？

可以用 `callbackFlow` 把多次回调转换成流，在 `awaitClose` 中释放监听。

下面假设 `source.observeSnapshots()` 提供的是**已经归并完成的、不可变的完整仓位快照**。订阅时它先发送当前快照，再按版本递增顺序串行回调后续快照，并返回一个可释放的订阅对象：

```kotlin
fun observePositionSnapshots(
    source: PositionSource
): Flow<PositionSnapshot> = callbackFlow {
    val subscription = source.observeSnapshots { snapshot ->
        // 使用合并缓冲；收集已取消时发送可能失败。
        trySend(snapshot)
    }

    awaitClose {
        subscription.dispose()
    }
}.buffer(Channel.CONFLATED)
```

同一份数据的新完整快照可以替代旧快照，因此这个展示接口允许合并。这里保留的是最后到达的值；如果源头允许乱序，就要先按版本归并，不能直接把最后到达的快照当成最新状态。

`Channel.CONFLATED` 只保留一个待接收值，新值会替换尚未接收的旧值。在通道仍开放时，`trySend()` 不会因为缓冲满而失败；它返回成功，也不表示每个快照都会交给下游。参见 [CONFLATED 缓冲语义](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel/-factory/-c-o-n-f-l-a-t-e-d.html)。

如果回调给的是“新增一笔成交”或“删除一个仓位”的增量，就不能照抄这里的合并策略。应先可靠归并业务数据，再输出状态。如果接收增量时采用有界缓冲且不允许丢弃，`trySend()` 可能因缓冲满而失败，必须检查结果，并按协议补拉或重新同步。

`callbackFlow` 是冷流，每次收集都会执行它的代码块。因此多个页面直接收集，可能注册多个监听；若注册监听同时创建连接，就可能重复建连。

可以在管理这份数据的 Repository 中用 `shareIn()` 或 `stateIn()` 共享上游，并明确作用域和启动策略。多个页面需要订阅同一个共享流实例；如果每次调用接口都重新创建共享流，仍会重复启动上游。

释放一个监听是否应该关闭 Socket，取决于连接是否还有其他使用者。`observeSnapshots()` 与 `dispose()` 的实现还必须正确处理注册、回调和取消并发发生的情况，参见 [callbackFlow API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/callback-flow.html)。

**面试时可以这样说：**

> 用 callbackFlow 把回调转换成流，用 awaitClose 释放订阅。callbackFlow 是冷流，多次收集可能重复注册，需要在合适的作用域共享。缓冲策略要区分完整状态和业务增量，不能为了流畅就丢关键更新。

## 4. SharedFlow 为什么会丢事件？加 extraBufferCapacity 就行吗？

考虑页面停止收集期间，ViewModel 发出一个提示：

```kotlin
private val _hints = MutableSharedFlow<UiHint>(
    replay = 0,
    extraBufferCapacity = 8,
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)
val hints = _hints.asSharedFlow()
```

这个配置可以暂存已订阅的收集者来不及接收的提示，缓冲满时丢弃最旧值。但**没有订阅者时，只保留 `replay` 指定的内容**。这里 `replay = 0`，页面回来后不会补收之前的提示，额外容量 8 不会变成离线收件箱。

默认 `MutableSharedFlow()` 没有 replay 和额外缓冲。存在订阅者时，`emit()` 可能挂起等待接收；不存在订阅者时会返回，值不会保存。`tryEmit()` 返回 true 也不能证明 UI 展示成功。语义见 [SharedFlow 的缓冲与无订阅者行为](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-shared-flow/)。

把 `replay` 改成 1，可以让新订阅者拿到缓存中的上一条事件，但也可能在旋转屏幕或重新进入页面时重复展示提示。`resetReplayCache()` 只是清缓存，不能代替对各个消费者处理结果的确认。

可丢失的临时提示，可以接受上述配置。订单结果、授权结果等需要可靠呈现的信息，应保存为可查询状态，不能只发一条瞬时通知。

**面试时可以这样说：**

> SharedFlow 的额外缓冲服务于活跃的慢订阅者，没有订阅者时只看 replay。此时 replay 为 0 不保留事件，改成 1 又可能在重新订阅时重复展示。可靠结果应有状态和确认机制，不能靠调大 buffer 保证送达。

![SharedFlow 没有订阅者时只保留 replay 内容，额外缓冲不能保存离线事件](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/kr2.png)

## 5. 页面怎么收集 Flow？多个流可以顺序 collect 吗？

Fragment 应在 `onViewCreated()` 等合适位置建立与 View 生命周期绑定的收集：

```kotlin
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        launch {
            viewModel.uiState.collect { render(it) }
        }
        launch {
            viewModel.hints.collect { showHint(it) }
        }
    }
}
```

两个 `launch` 分别收集两个流。如果直接先 `uiState.collect` 再 `hints.collect`，StateFlow 正常情况下不会结束，后一个收集就无法开始。

View 的生命周期低于 `STARTED` 时，上面的内部任务会取消；重新达到 `STARTED` 或更高状态后，会重新启动收集。使用 `viewLifecycleOwner` 可以避免 Fragment 还存在、View 已经销毁时继续渲染旧 View。参见 [Android View 生命周期协程指南](https://developer.android.com/topic/libraries/architecture/views/coroutines-views)。

Compose 可以用 `collectAsStateWithLifecycle()` 把状态转成生命周期感知的 Compose State。临时事件的收集仍然需要明确生命周期、消费方式和是否允许丢失。

取消收集会取消当前收集协程及其子任务。在独立作用域持续运行的热流生产者，不会因此自动停下；如果希望没有订阅者时停止共享上游，还需要配合 `WhileSubscribed` 等策略，并实现资源清理。

**面试时可以这样说：**

> Fragment 用 View 生命周期配合 repeatOnLifecycle。多个不会自行结束的流分别 launch 收集，不能顺序等待。页面停止收集与生产者停止是两件事，还要明确由谁管理上游资源。

## 6. MVVM 和 MVI 在使用场景上有什么区别？

MVVM 主要描述 View、ViewModel、Model 的职责划分。ViewModel 对外提供状态与操作入口，View 负责展示和传递交互。它可以配合双向绑定，也可以采用单向数据流，并不要求 UI 直接修改业务状态。

MVI（Model–View–Intent）在 Android 项目里通常强调：交互和外部输入作为 Intent 或 Action 进入统一流程，由状态更新逻辑产出新 State，再由 UI 渲染。这里的 Intent 表示用户意图或业务输入，与 `android.content.Intent` 是两个概念。

```text
用户操作 / 外部输入
→ Action
→ 处理逻辑与副作用
→ Result
→ 状态归并
→ State
→ UI 渲染
```

这里的“副作用”指网络请求、数据库写入等与外部交互的操作；纯 reducer 则只根据旧状态和输入计算新状态。

团队对 MVI 的具体实现并不完全一致：可以用 Store 集中管理状态，也可以用状态机约束转换，或把这些逻辑放在 AndroidX ViewModel 里。因此，项目可以沿用 MVVM 的职责划分，同时采用 MVI 风格的单向状态更新。

简单展示页，用少量状态和方法就能表达清楚，没必要为每个点击都增加多层包装。仓位页面有刷新、推送、筛选、平仓、错误恢复等并发输入时，统一状态转换更容易检查冲突和编写测试，但也会增加 Action、Result 和状态建模成本。

Android 官方推荐的单向数据流原则，可以用来指导这两类实现；这不等于官方要求项目必须叫 MVI。参见 [UI 层与单向数据流](https://developer.android.com/topic/architecture/ui-layer)。

**面试时可以这样说：**

> MVVM 划分职责，MVI 更强调统一输入和可追踪的状态转换，两者可以结合。简单页面保持轻量，输入来源多、状态转换复杂的页面可以加强 reducer 或状态机约束，关键是控制状态写入规则。

## 7. “MVVM 数据倒灌”是什么意思？怎么解决？

“数据倒灌”不是一个定义统一的 Android API 术语，需要结合现象确认。常见情况有下面几种。

**第一种是渲染触发回写。** UI 根据 ViewModel 的状态更新输入框，`setText()` 又触发 TextWatcher；监听器把这次程序更新当成用户输入，写回 ViewModel，甚至再次发起请求。

可以在渲染前比较新旧值，并在程序更新期间暂停对应监听，或用标记跳过这次回调，避免把页面渲染当成用户操作。真正的用户输入通过 `onQueryChanged()` 等入口提交，由 ViewModel 判断是否需要更新。`distinctUntilChanged()` 能减少相同值传播，但如果不同层反复把同一个输入格式化成不同的值，还需要统一格式化规则和处理位置。

**第二种是旧结果覆盖新状态。** 用户先搜 A 再搜 B，B 先返回，A 后返回，把页面又改成 A。可采用取消旧请求、请求版本校验，或按查询条件切换流；如果无法保证旧请求取消，就要在提交结果前检查请求标识，确认它仍对应当前查询。

**第三种是重建后旧缓存重新出现。** 页面重新订阅获得 StateFlow 保存的当前值，本身符合状态重放语义，但这份值可能已经过时。应在 State 中表达当前账户、数据版本和新鲜度，刷新期间按产品要求展示缓存或加载态。切换账户时，还应清理或隔离旧账户状态，拒绝旧账户请求的回写，避免展示其他账户的数据。

迁移到 MVI 并不会自动解决这些问题。状态更新时不检查请求版本，旧结果仍可能覆盖新状态；UI 渲染时又触发相同的输入 Action，则可能形成回写环路。

**面试时可以这样说：**

> 我会先确认倒灌是渲染回写、旧请求覆盖，还是缓存重放。对应的处理方式是分离输入与渲染、校验请求标识，以及记录数据版本和是否过期。问题在数据流和更新规则，不能只靠改架构名字解决。

## 8. 页面想刷新数据，在 MVI 里怎么做？

UI 发送 `Refresh` Action，状态持有者安排请求，Repository 合并结果，再通过 State 驱动页面。UI 不直接把新列表写回业务状态，也不需要绕回全局总线发送“请刷新”。

下面是一个简化实现：**刷新过程中忽略重复刷新**，保留已有列表，网络 I/O 失败时结束刷新并显示错误。业务数据统一由 Repository 管理。

```kotlin
sealed interface PositionsAction {
    object Refresh : PositionsAction
}

data class RefreshStatus(
    val isRefreshing: Boolean = false,
    val error: String? = null
)

data class PositionsUiState(
    val rows: List<PositionRow> = emptyList(),
    val isRefreshing: Boolean = false,
    val error: String? = null
)

class PositionsViewModel(
    private val repository: PositionsRepository
) : ViewModel() {
    private val refreshStatus = MutableStateFlow(RefreshStatus())

    val uiState: StateFlow<PositionsUiState> = combine(
        repository.rows,
        refreshStatus
    ) { rows, status ->
        PositionsUiState(rows, status.isRefreshing, status.error)
    }.stateIn(
        viewModelScope,
        SharingStarted.WhileSubscribed(5_000),
        PositionsUiState()
    )

    @MainThread
    fun onAction(action: PositionsAction) {
        when (action) {
            PositionsAction.Refresh -> refresh()
        }
    }

    private fun refresh() {
        if (refreshStatus.value.isRefreshing) return
        refreshStatus.value = RefreshStatus(isRefreshing = true)

        viewModelScope.launch {
            try {
                repository.refreshPositions()
            } catch (e: CancellationException) {
                throw e
            } catch (e: IOException) {
                refreshStatus.update { it.copy(error = "刷新失败，请重试") }
            } finally {
                refreshStatus.update { it.copy(isRefreshing = false) }
            }
        }
    }
}
```

这段代码有几项约定需要说清：

- `onAction()` 由主线程调用，因此检查和设置刷新标记之间没有并行调用。`@MainThread` 是调用约定，不会自动切线程。
- Repository 通过只读 StateFlow 暴露不可变的 `rows` 列表快照；`refreshPositions()` 可以从主线程调用，内部负责耗时工作，并在返回前按版本规则提交结果。
- HTTP 快照与 Socket 更新在 Repository 统一归并，不能在刷新返回时盲目覆盖较新的推送。
- 这里演示网络 I/O 异常；业务拒绝、鉴权失败等结果应按项目错误模型补充状态，取消异常继续抛出。
- `WhileSubscribed(5_000)` 表示最后一个订阅者离开 5 秒后停止上游收集，默认仍保留缓存。再次进入页面时可能先收到缓存值，然后才收到重新计算的状态；5 秒不是数据有效期。参见 [WhileSubscribed API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-sharing-started/-companion/-while-subscribed.html)。
- 页面停止收集，不会取消这里由 `viewModelScope` 启动的独立刷新任务；ViewModel 清除时才会取消该作用域。是否在离开页面时取消刷新，要另定策略。

这个示例只在收到 `Refresh` 时发起请求，收集 `uiState` 本身不会自动刷新。如果首屏需要加载，应在约定的初始化入口触发，并区分“尚未加载”和“加载后确实为空”。

`combine` 会根据两个流各自的最新值生成状态，但列表与刷新标记来自两个独立的流，UI 可能先观察到其中一个变化。如果业务要求二者作为同一次状态更新出现，应统一归并为一个状态后发布。

这是把状态更新集中到 ViewModel 的轻量实现。如果团队使用严格的 reducer，可以把开始、成功和失败建模为 Result，再交给纯函数转换刷新状态；网络请求仍放在副作用处理层。

**面试时可以这样说：**

> UI 发 Refresh，状态先进入刷新中，副作用执行请求，结果通过统一状态入口更新。需要明确重复刷新策略、失败和取消处理，并让查询与推送在 Repository 按版本归并，不能各自改一份列表。

![MVI 风格刷新流程：UI 提交 Refresh，Repository 归并业务数据，ViewModel 生成 uiState 供页面渲染](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/kr3.png)

图中的“刷新状态变化”可以先于网络结果传到 UI；普通 Socket 推送只更新业务数据，不会自行结束正在进行的刷新。

## 9. 多个 Intent 同时进来，MVI 就能保证顺序吗？

即使 Action 按顺序进入同一个入口，异步任务也可能乱序完成。如果每个 Action 进来就 `launch` 请求，仍然可能出现先发后到。

应按操作含义选择策略：

| 输入 | 可采用的策略 |
| --- | --- |
| 搜索变化 | 切换到新查询 |
| 重复刷新 | 忽略重复或重启 |
| 顺序修改 | 串行或版本约束 |
| 独立加载 | 并发后分别归并 |

`flatMapLatest()` 适合切换查询等可取消工作，不能随意用于已经发给服务端的业务写操作。客户端取消等待，不代表服务端撤销执行；这类操作要靠请求 ID、幂等性和结果查询确认。

即使所有 reducer 都在主线程同步运行，异步 Result 仍需要携带请求标识或数据版本，才能判断它是否还属于当前状态。

把每个 Intent 都塞进一个 Channel，也不能省略容量和过载策略；队列无限增长时，用户操作依然会越来越晚得到处理。

**面试时可以这样说：**

> MVI 让输入和状态转换集中可见，但不会自动消除异步乱序。查询、刷新、业务写操作要采用不同并发策略，返回结果带身份和版本，队列也要有容量边界。

## 10. Toast、导航、支付结果这种一次性事件怎么处理？

先确定丢失或重复的代价。

页面不可见时可以忽略的临时提示，可以用 `replay = 0` 的 SharedFlow，并接受停止收集期间可能错过提示的行为。由当前按钮点击直接触发的导航，也可以由 UI 根据明确的交互和状态处理，不必让所有导航都经过事件总线。

订单或支付结果需要可靠确认，应在状态中保留业务结果或待确认事项，UI 展示后再回传确认。对于多个结果，需要用带 ID 的集合或队列，单个 `pendingResult` 字段可能被后来的结果覆盖。

```text
业务状态保存结果 resultId
→ UI 根据状态展示
→ UI 回传 Acknowledge(resultId)
→ 状态持有者移除对应待展示项，保留业务结果
```

确认时按 ID 匹配，避免旧页面的确认误清掉新结果。这个确认只表示 UI 已处理展示事项，不代表服务端业务成功，也不应删除订单或支付记录。是否要在进程重建后恢复，决定了待展示事项是否需要持久化。Android 官方对 ViewModel 产生的 UI 行为，推荐通过状态更新表达，参见 [UI events 指南](https://developer.android.com/topic/architecture/ui-layer/events)。

Channel 也不能自动保证“恰好消费一次”。元素被取出后，消费者可能在展示前取消；展示后、确认前进程也可能退出。需要结合持久化、幂等处理和恢复规则，接受并处理可能的重试或重复展示。

因此，“用 Channel 就不会丢失或重复处理事件”不成立。对于导航，还应检查当前路由和业务流程，避免恢复时把用户再次带到不合适的页面。

**面试时可以这样说：**

> 可丢提示可以用瞬时事件，重要业务结果应该可查询、可恢复，并通过 ID 确认处理。SharedFlow 和 Channel 都不能单独保证 UI 恰好执行一次，需要考虑取消、重复订阅和进程退出。

## 11. StateFlow 明明改了数据，页面为什么没刷新？

先检查是否原地修改了对象：

```kotlin
// 错误示例：假设这里的 rows 被声明为 MutableList<PositionRow>。
// 它与前文使用 List 的 PositionsUiState 不同。
state.value.rows.add(newRow)
state.value = state.value
```

StateFlow 按 `equals()` 判断值是否变化。上面先原地修改列表，再赋回同一个对象，不会触发新的状态发射。应使用不可变状态语义，创建新列表和必要的新 Item，并只向外暴露只读 StateFlow。

```kotlin
_state.update { old ->
    old.copy(rows = old.rows + newRow)
}
```

`List<T>` 是只读接口，不等于底层对象一定不可变；如果其他地方仍然持有可变集合的引用并修改它，问题仍在。`data class.copy()` 也是浅拷贝，内部可变对象不会自动复制。

多线程更新依赖旧值时，`update` 可以原子地完成“读取旧值、计算新值、写回”的过程。但它的 lambda 在并发竞争下可能执行多次，因此里面应只计算新状态，不能发请求、写数据库或执行其他必须只做一次的业务操作。参见 [MutableStateFlow.update API](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/update.html)。

如果数据对象正确，再检查收集是否处于活跃生命周期、上游有没有被取消，以及 UI 是否只观察了部分派生字段。StateFlow 的线程安全也不代表内部的可变对象线程安全。

**面试时可以这样说：**

> 先看有没有原地修改对象、equals 是否认为相等，再看生命周期和派生状态。用不可变状态和 update 表达变化，update 里保持纯计算，不把流本身线程安全理解成对象内部也安全。

## 12. 怎么测试这套架构，避免只是在画架构图？

先测试状态规则，再测试异步和生命周期行为。

对纯 reducer，输入旧 State 和 Result，检查新 State 是否符合预期。例如刷新失败后保留原有 `rows`，同时结束刷新；过期请求结果不修改当前查询的状态；旧确认 ID 不清掉新的待处理事项。

对 ViewModel，用可控制返回顺序的 fake Repository（测试替身），验证重复刷新策略、取消传播和错误状态。使用协程测试调度器控制时间，避免测试依赖真实等待。参见 [Android Flow 测试指南](https://developer.android.com/kotlin/flow/test)。

HTTP 快照与 Socket 推送的冲突归并由 Repository 负责，应为真实 Repository 注入可控的数据源，单独验证旧版本丢弃、账户切换和结果合并。fake Repository 可以帮助测试页面反应，但不能证明真实归并逻辑正确。

`stateIn()` 配合 `WhileSubscribed` 时，还有一个测试细节：没有活跃订阅者，上游可能根本没有启动。读取 `.value` 不会启动上游收集，测试需要建立收集任务，再推进调度器并验证结果。

最后验证资源生命周期：两个消费者是否复用同一个共享上游，最后一个消费者离开后是否按约定延时释放监听，还有其他消费者时是否继续保留所需连接。再检查旋转屏幕后是否重复导航，以及切换账户后旧事件是否还会影响新页面。

**面试时可以这样说：**

> reducer 测状态转换，ViewModel 测异步顺序和错误，生命周期测试订阅与资源释放。使用 WhileSubscribed 时要让测试真的开始收集，最终验证可观察状态和资源数量，而不只是某个方法被调用。

---

回答架构题时，可以结合具体业务说明：哪一层统一管理数据、谁能修改状态、如何拒绝过期结果，以及页面离开后的任务和结果该怎样处理。这些规则明确后，再说明为什么选择相应的通信工具和架构方式。

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**
