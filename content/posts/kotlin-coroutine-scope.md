---
title: "Android 中 viewModelScope、lifecycleScope、GlobalScope：协程作用域怎么选？"
date: 2026-06-16T20:00:00+08:00
draft: false
categories: ["Android"]
tags: ["Android", "Kotlin", "协程"]
---

协程不是「开个线程跑一下」就完事——**谁启动、谁负责取消、子协程失败会不会拖垮兄弟**，都挂在「作用域」上。搞懂作用域，结构化并发才算真的入门。

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/scope0.png)

## 一、CoroutineScope 到底是什么

```kotlin
public interface CoroutineScope {
    public val coroutineContext: CoroutineContext
}
```

`CoroutineScope` 只有一个属性：`coroutineContext`。平时说的「作用域」，本质是 **Context 里挂着 Job（或 SupervisorJob）+ 调度器** 的一包环境。`launch` / `async` 都是 `CoroutineScope` 的扩展函数：

```kotlin
fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job
```

你在哪个 `CoroutineScope` 上调用 `launch`，新协程就挂到哪个 Job 树下面，成为**子 Job**。

> **核心公式：CoroutineScope = Context + Job**
>
> - **Context**（含 Dispatcher）决定协程**在哪条线程跑、能访问哪些数据**（ThreadLocal 等）
> - **Job** 决定协程的**生命周期**——何时启动、何时取消、父子关系
>
> 两者结合，就是一个完整的协程运行环境。后面讲的各种 scope，本质上都是这两要素的不同组合。

## 二、Job 树与结构化并发

```
父 Job (viewModelScope)
├── 子 Job A (launch 请求用户)
├── 子 Job B (launch 请求配置)
└── 子 Job C (async 解析 JSON)
```

规则可以记成三句：

1. **父协程的 Job 在所有子 Job 完成前不会变为 completed**——这是 Job 状态机的内置保证，无需手动 `join`。
2. **子协程失败**，默认会**取消父协程**；父取消会**向下取消**所有子 Job。
3. **父取消**（例如 `viewModelScope` 在 `onCleared` 里 cancel）会**向下取消**整棵子树。

这就是 **Structured Concurrency（结构化并发）**：生命周期有边界，不会冒出「没人管的 GlobalScope 协程」。

```kotlin
viewModelScope.launch {
    val a = async { repo.fetchA() }
    val b = async { repo.fetchB() }
    combine(a.await(), b.await()) { /* ... */ }
}
// ViewModel 销毁 → viewModelScope 取消 → 上面 launch 及其所有 async 子协程一并取消
```

## 三、常见作用域一览

| 作用域 | 类型 | 典型用途 | 生命周期 |
|---|---|---|---|
| `GlobalScope` | 单例 `CoroutineScope` | 几乎不应使用 | 进程级 |
| `MainScope()` | 自定义，主线程 + `SupervisorJob` | 非 Android 的主线程 UI 协程 | 手动 `cancel()` |
| `coroutineScope { }` | 挂起函数，临时子树 | 并发子任务，一个失败全取消 | 块结束即完成 |
| `supervisorScope { }` | 挂起函数，SupervisorJob | 并行任务互不影响 | 块结束即完成 |
| `withContext(Dispatchers.IO) { }` | 挂起函数，继承父 Job | 切换线程，不新建 Job 子树* | 块结束即恢复 |
| `viewModelScope` | ViewModel 扩展 | ViewModel 内启动协程 | `onCleared()` |
| `lifecycleScope` | Activity / Fragment 扩展 | UI 生命周期内任务 | `Lifecycle` DESTROYED |
| `rememberCoroutineScope()` | Compose | 与 Composable 绑定的手动 scope | Composable 离开组合 |

\* `withContext` 不创建新 Job——它继承父协程的 Job，仅切换调度器/协程上下文。块内代码仍是原来那棵 Job 树的一部分。

## 四、GlobalScope：知道存在，尽量别用

```kotlin
GlobalScope.launch {
    // 与任何 UI / ViewModel 生命周期无关
    delay(60_000)
    saveToDb()
}
```

`GlobalScope` 的 `coroutineContext` 是 `EmptyCoroutineContext`（没有 Job 父节点意义上的结构化约束）。子协程异常可能变成**未捕获异常**，且 Activity 销毁后协程仍在跑——**内存泄漏**的经典来源。

官方态度很明确：应用代码里用 **带生命周期的 scope**（`viewModelScope`、`lifecycleScope`、自己 `CoroutineScope(SupervisorJob() + Dispatchers.Main)` 并在适当时 `cancel()`）。

## 五、挂起作用域：coroutineScope 与 supervisorScope

两者都是**挂起函数**，会建一棵**新的 Job 子树**，并在块内所有子协程结束后才返回。

### coroutineScope —— 一个失败，全家取消

```kotlin
suspend fun loadHome() = coroutineScope {
    val user = async { api.getUser() }
    val feed = async { api.getFeed() }
    HomeUi(user.await(), feed.await())
}
```

任一 `async` 失败 → scope 失败 → 兄弟协程被取消 → 异常向上抛给 `loadHome` 的调用方。

### supervisorScope —— 一个失败，兄弟继续

```kotlin
suspend fun loadHomeResilient() = supervisorScope {
    // supervisorScope 阻止失败取消兄弟，但 await() 仍会拿到异常 → runCatching 把失败转为 null
    val user = async { runCatching { api.getUser() }.getOrNull() }
    val feed = async { api.getFeed() }  // 即使用户挂了，feed 仍会跑完
    HomeUi(user.await(), feed.await())
}
```

底层是 `SupervisorJob`：子协程异常**不会**取消兄弟。适合「多个独立请求，允许部分失败」的场景。

> **注意**：`supervisorScope` 阻止的是异常在 Job 树上的**取消传播**，但异常本身并没有消失——如果你不在 `async` 内部用 `runCatching` 捕获，调用 `await()` 时异常仍会抛出，可能导致上层逻辑中断。因此最佳实践是：**在 `supervisorScope` + `async` 里，要么在 `await()` 处用 try-catch 处理，要么像上面一样用 `runCatching` 在子协程内部消化掉。**

细节可参考 [async 异常那篇]({{< relref "kotlin-coroutine-async-exception.md" >}})。

### 和 withContext 的分工

用两个场景区分就很清楚：

- **`withContext`**：同一个任务里，「先去 IO 线程拿数据，再回主线程解析」——串行，换线程不换 Job。
- **`coroutineScope`**：同一时刻，「同时调 A 接口和 B 接口，两个请求平级」——并行，建一棵新 Job 子树。

```kotlin
// withContext：串行切换线程，仍是同一条协程
suspend fun fetchAndParse() = withContext(Dispatchers.IO) {
    val raw = download(url)
    parse(raw)  // 仍在同一结构化子树里，父 cancel 会取消这里
}

// coroutineScope：并行拆多个子任务
suspend fun loadHome() = coroutineScope {
    val user = async { api.getUser() }
    val feed = async { api.getFeed() }
    HomeUi(user.await(), feed.await())
}
```

`withContext` **不**为了「新建一棵并发子树」——它主要做**线程 / 上下文切换**。并发拆多个子任务用 `coroutineScope { async { } }`，别指望单用一个 `withContext` 替代整段并行逻辑。

## 六、Android 里与生命周期绑定的 Scope

### viewModelScope

```kotlin
class ProfileViewModel : ViewModel() {
    fun refresh() {
        viewModelScope.launch {
            _state.value = UiState.Loading
            _state.value = UiState.Success(repo.loadProfile())
        }
    }
}
// onCleared() → viewModelScope.coroutineContext[Job]?.cancel()
```

配置变更旋转屏幕时 ViewModel 还在，scope 还在；用户退出页面、`ViewModel` 被清掉时，正在跑的请求会被 cancel。不必自己持有一个 `Job` 再 `cancel()`。

### lifecycleScope

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { render(it) }
            }
        }
    }
}
```

`lifecycleScope` 在 `Lifecycle` 到达 **DESTROYED** 时取消。比「在 Activity 里裸 `GlobalScope.launch`」安全一个数量级。

### Compose：rememberCoroutineScope

```kotlin
@Composable
fun SnackbarHost() {
    val scope = rememberCoroutineScope()
    Button(onClick = {
        scope.launch {
            snackbarHostState.showSnackbar("已保存")
        }
    }) { Text("保存") }
}
```

这个 scope 的 Job 父节点来自 Compose 提供的 `CoroutineScope`，**不会**随旋转自动等同于 `viewModelScope` 那么长——它跟着 Composable 走，适合一次性 UI 副作用（动画、Snackbar），长期网络请求仍应放在 ViewModel。

> `rememberCoroutineScope` 主要用于**响应用户交互**（如点击按钮）时启动协程。如果是在 Composable **进入组合时就自动执行**的任务（如初始化数据、订阅 Flow），优先考虑 `LaunchedEffect`。

## 七、自己造 Scope：Application 级与进程内单例

```kotlin
class App : Application() {
    val applicationScope = CoroutineScope(
        SupervisorJob() + Dispatchers.Default
    )
}
```

`SupervisorJob` 表示某个子任务失败不拖垮兄弟。进程被系统杀死时协程自然消亡，没有泄漏问题——和 ViewModel / Activity 不同，它们会因配置变更或返回键而销毁但进程还活着，不 cancel 才真会漏。所以 Application 级 scope **不需要特意 `cancel()`**。

数据库迁移、日志上报等**短、可控**的后台任务可以放这里，仍然不要塞长生命周期 UI 逻辑。

## 八、取消是怎么传下去的

```kotlin
viewModelScope.launch {
    coroutineScope {
        launch { delay(10_000); save() }
        launch { delay(500); throw IOException("网络错误") }
    }
}
```

内层 `coroutineScope` 里第二个 `launch` 失败 → 整个 `coroutineScope` 失败 → 外层 `viewModelScope.launch` 失败 → 第一个 `launch` 也被取消。

手动取消同样向下传播：

```kotlin
val job = viewModelScope.launch {
    repeat(1000) {
        delay(500)
        println("tick $it")
    }
}
// 某处
job.cancel()
```

> **关键前提**：取消是**协作式（cooperative）**的——协程只有在到达挂起点（`delay`、`withContext` 等）或主动检查 `isActive` 时才会真正终止。纯 CPU 计算的 `while` 循环里如果不调 `ensureActive()` 或检查 `isActive`，`cancel()` 发了也不会停。

## 九、选型速查

| 需求 | 推荐方案 | 核心逻辑 |
|---|---|---|
| ViewModel 内数据请求 / 状态维护 | `viewModelScope` | 随 ViewModel 生命周期自动取消 |
| UI 生命周期绑定（监听数据变化等） | `lifecycleScope` + `repeatOnLifecycle` | 后台暂停、前台恢复，DESTROYED 时自动取消 |
| 并行子任务，要求「全有或全无」 | `coroutineScope { async { } }` | 结构化并发，任一失败全部取消 |
| 并行子任务，允许部分失败 | `supervisorScope { async { } }` | 异常隔离，子任务失败不影响兄弟 |
| 仅在特定线程执行阻塞 / 耗时操作 | `withContext(Dispatchers.IO) { }` | 线程切换，不改变当前 Job 树结构 |
| 进程级、脱离生命周期的后台任务 | 自建 `CoroutineScope(SupervisorJob() + …)`，手动 `cancel()` | 完全手动管理生命周期 |
| 随手 `GlobalScope.launch` | 别 | —— |

## 十、和异常、async 的关系

作用域决定**谁活着、谁被取消**；`async` 的异常何时抛出，由 `await` 决定——那是另一个维度。

---

**一句话**：作用域 = **Job 树 + Context**；选 scope 就是选**这棵树的边界**。子协程挂在哪棵树下，就受哪条「父等子、失败传播、取消向下」的规则约束。Android 开发里默认优先 **`viewModelScope` / `lifecycleScope`**，临时并发用 **`coroutineScope` / `supervisorScope`**，避开 **`GlobalScope`**。

*基于 kotlinx.coroutines 1.7+ 与 AndroidX Lifecycle 常见用法整理，如有疏漏欢迎指正。*

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**
