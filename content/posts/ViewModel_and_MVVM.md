---
title: "Android 开发里如何理解 ViewModel？"
date: 2026-06-06T10:00:00+08:00
draft: false
aliases: ["/posts/viewmodel-blog/"]
categories: ["Android"]
tags: ["Android", "ViewModel"]
---

> 一篇关于 Android ViewModel 的学习笔记——它的身世、它解决的问题，以及那些被名字带偏的认知。

![封面图](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/viewmodel0.png)

---

## 一、名字里的误会

一个常见场景：聊到项目架构，对方问"你们用 MVVM 吗？"——"用了用了，我们用了 ViewModel。"

这个对话太熟悉了，我自己也这么说过。但后来翻文档、看官方说明，才发现这个回答其实把两个东西混在了一起：**Jetpack ViewModel** 和 **MVVM 模式里的 ViewModel**。

名字一样，不代表是同一个东西。

这篇文章试着把 ViewModel 的来龙去脉理一遍：它到底解决什么痛点、为什么会叫这个名字、实际应该怎么用。水平有限，难免疏漏，欢迎指正。

---

## 二、ViewModel 的身世：它到底为解决什么问题而来

### 当时的痛点

2017 年之前，Android 开发有一个很头疼的问题：**屏幕旋转导致 Activity 重建，数据全丢**。

当时的应对方案五花八门：

- `onRetainNonConfigurationInstance()` —— 手动管理对象，类型不安全，容易出错
- `Fragment.setRetainInstance(true)` —— 能用，但跟 Fragment 生命周期绑得太紧
- 自己维护全局 Map 存数据 —— 内存泄漏温床

每种方案都"能跑"，但都不优雅。

### Architecture Components 的诞生

2017 年 Google I/O，Android 团队发布了 **Architecture Components**（后来并入 Jetpack），其中包含了四个核心组件：

| 组件 | 职责 |
|---|---|
| **ViewModel** | 在配置变更中保持 UI 数据存活 |
| **LiveData** | 生命周期感知的可观察数据持有者 |
| **Room** | SQLite 的抽象层 |
| **Lifecycle** | 生命周期感知的基础设施 |

看到了吗——ViewModel 的出生证明上写的不是"MVVM 的 VM"，而是"在配置变更中保持 UI 数据存活"。

### 官方怎么说

Android 团队在多处提到过：ViewModel 这个名字确实容易造成误解。官方文档和演讲中多次说明，ViewModel 更多是"为 View 准备数据的 Model"——它跟 MVVM 模式里那个 ViewModel 在概念上有重叠，但不是同一个设计意图的产物。

> **引用说明**：以上关于 Android 团队的表述来自官方文档和演讲的总体精神，非逐字引用。如有不准确，欢迎指正。

所以结论是：ViewModel 的定位一直很清楚——**它是一个生命周期感知的、跨越配置变更存活的数据容器**。至于 MVVM，那是另一个话题。

---

## 三、"ViewModel 是不是 MVVM 的 VM？"——严格来说不是

先看 MVVM 模式的理论定义，再跟 Jetpack ViewModel 放在一起比：

| | MVVM 理论中的 ViewModel | Jetpack ViewModel |
|---|---|---|
| **目的** | 将 Model 数据适配为 View 可用的格式，处理展示逻辑 | 在配置变更中保持 UI 状态存活 |
| **与 View 的关系** | View 通过数据绑定观察 ViewModel，ViewModel 不知道 View 存在 | 不持有 View 引用，通过 LiveData / Flow / StateFlow 暴露数据给 View |
| **生命周期** | 随 View 的创建和销毁 | 比 Activity / Fragment 长，直到 `ViewModelStoreOwner` 真正 finish |
| **与 Model 的关系** | 调用业务逻辑、转换数据给 View | 通常持有 Repository / UseCase 引用，但不是强制的 |

关键点是：**Jetpack ViewModel 是一个架构组件，不是一种架构模式**。

可以在 MVP 里用它：

```kotlin
class MyPresenter(private val repository: MyRepository) {
    // Presenter 持有逻辑，ViewModel 只存数据
}

class MyViewModel : ViewModel() {
    private val _state = MutableStateFlow(UiState())
    val state: StateFlow<UiState> = _state.asStateFlow()
    // ViewModel 负责存活，Presenter 负责逻辑
}
```

也可以在 MVI 里用它：

```kotlin
class MyViewModel : ViewModel() {
    private val _state = MutableStateFlow(MyUiState())
    val state: StateFlow<MyUiState> = _state.asStateFlow()

    fun dispatch(intent: MyIntent) {
        // 处理 Intent，更新 State
    }
}
```

当然也可以在 MVVM 里用它——但"用了 ViewModel 就等于用了 MVVM"这个等式不成立。ViewModel 只是工具箱里的一个工具，不是一套完整的架构理念。

---

## 四、ViewModel 到底"是"什么？——三个核心能力

放下 MVVM 的包袱之后，重新看 ViewModel，它其实就做了三件事：

### 1. 配置变更存活

这是 ViewModel 的核心价值。屏幕旋转、语言切换、系统深色模式切换——这些配置变更会重建 Activity，但 ViewModel 不受影响。

```kotlin
class ProfileViewModel : ViewModel() {
    // 这个数据在屏幕旋转后依然存在
    var userName by mutableStateOf("")
        private set

    fun loadProfile() {
        viewModelScope.launch {
            // 旋转屏幕不会取消这个请求
            val profile = repository.getProfile()
            userName = profile.name
        }
    }
}
```

**原理**：ViewModel 存储在 `ViewModelStore` 里，`ViewModelStore` 的持有者是 `ViewModelStoreOwner`（Activity / Fragment 实现了这个接口）。配置变更时 Activity 重建，但 `ViewModelStoreOwner` 还是同一个，所以 ViewModel 保留了下来。直到 Activity 真正 finish（用户返回、手动 `finish()` 等），`ViewModelStore` 才会被清空。

### 2. 生命周期感知的协程作用域

```kotlin
class MyViewModel : ViewModel() {
    fun fetchData() {
        viewModelScope.launch {
            // 当 ViewModel 被清除时，这个协程自动取消
            val data = repository.getData()
            _state.value = data
        }
    }
}
```

`viewModelScope` 是 ViewModel 提供的协程作用域，在 `ViewModel.onCleared()` 时自动取消所有正在运行的协程。意味着不用手动管理 Job、不用担心 Activity 销毁后协程还在跑导致的内存泄漏。

### 3. 同一 Activity 内 Fragment 间的数据共享

```kotlin
// FragmentA
class FragmentA : Fragment() {
    private val sharedViewModel: SharedViewModel by activityViewModels()
    // ...
}

// FragmentB
class FragmentB : Fragment() {
    private val sharedViewModel: SharedViewModel by activityViewModels()
    // 拿到的是同一个 ViewModel 实例
}
```

这在多 Fragment 需要共享数据时很有用——比如一个列表页和一个详情页，共享选中项的状态。

---

## 五、常见误用——可能比"误解"本身更值得记录

抛开名字的包袱，日常使用里常见这几个坑：

### 1. 在 ViewModel 里持有 Context / View 引用

```kotlin
// ❌ 错误：Activity 旋转后这个 Context 已经失效
class MyViewModel(private val context: Context) : ViewModel() {
    fun getString(@StringRes resId: Int) = context.getString(resId)
}
```

ViewModel 比 Activity 活得久。持有旧的 Activity Context 会导致内存泄漏。如果需要 Context，用 `AndroidViewModel`（持有 Application Context），或者通过 `SavedStateHandle` 传递需要的参数。

```kotlin
// ✅ 正确
class MyViewModel(application: Application) : AndroidViewModel(application) {
    fun getString(@StringRes resId: Int) = getApplication<Application>().getString(resId)
}
```

### 2. 只用 ViewModel 处理配置变更，忽略了进程死亡

配置变更存活 ≠ 进程死亡存活。App 切到后台被系统杀死再恢复时，ViewModel 不会自动恢复数据。这时候需要 **SavedStateHandle**：

```kotlin
class MyViewModel(private val savedStateHandle: SavedStateHandle) : ViewModel() {
    val searchQuery: String
        get() = savedStateHandle.get<String>("search_query") ?: ""

    fun onSearchQueryChanged(query: String) {
        savedStateHandle["search_query"] = query
    }
}
```

`SavedStateHandle` 的原理是利用了 Activity 的 `onSaveInstanceState` 机制，在进程被杀死时序列化数据，恢复时反序列化。这个机制有数据量限制（约 1MB），大数据还是得持久化。

### 3. ViewModel 越来越重

刚开始用的时候 ViewModel 很轻，写着写着各种逻辑全往里塞——网络请求、数据转换、业务规则、甚至 UI 状态机。最后发现：

```kotlin
class HeavyViewModel(
    private val repo1: Repo1,
    private val repo2: Repo2,
    private val repo3: Repo3,
    private val useCase1: UseCase1,
    private val useCase2: UseCase2,
    // ... 参数列表比构造函数还长
) : ViewModel() {
    // 几百行业务逻辑混在一起
}
```

ViewModel 本质是**数据持有 + 协调**，不是逻辑仓库。复杂逻辑抽出去：

```
ViewModel  →  持有 UI 状态 + 调用 UseCase / Repository
UseCase    →  单一职责的业务逻辑
Repository →  数据来源协调（本地 / 远程）
```

---

## 六、Compose 时代的 ViewModel

Compose 里获取 ViewModel 的方式跟 View 系统类似：

```kotlin
@Composable
fun ProfileScreen(
    viewModel: ProfileViewModel = hiltViewModel()
) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    ProfileContent(
        state = state,
        onRefresh = viewModel::refresh
    )
}
```

几个值得注意的点：

### ViewModel 和 Compose 自带的状态管理不冲突

Compose 有 `remember`、`rememberSaveable` 等状态管理工具，它们解决的是**Composable 内部**的状态问题。ViewModel 解决的是**跨 Composable 的、与生命周期对齐的**状态问题。两者是互补的，不是替代关系。

### `collectAsStateWithLifecycle()` 而不是 `collectAsState()`

```kotlin
// ⚠️ 不够好：Activity 在后台时流仍然在收集
val state by viewModel.state.collectAsState()

// ✅ 推荐：生命周期感知的收集，后台停止更新
val state by viewModel.state.collectAsStateWithLifecycle()
```

这是 Compose 里一个容易忽视的细节。`collectAsStateWithLifecycle()` 来自 `lifecycle-runtime-compose`，会在生命周期低于 STARTED 时停止收集上游流，减少不必要的重组和资源消耗。

### ViewModel 与 Navigation Compose 的配合

```kotlin
NavHost(navController, startDestination = "profile") {
    composable("profile") {
        // 每个导航目标有自己的 ViewModel
        val viewModel: ProfileViewModel = hiltViewModel()
        ProfileScreen(viewModel)
    }
}
```

ViewModel 的 scope 跟导航图的 back stack entry 绑定——进入时创建，弹出时清除。这个粒度恰好适合单屏的状态管理。

---

## 七、一点个人理解

回顾下来，ViewModel 这个名字确实带偏了很多人。它本质上是一个**配置变更存活的数据容器 + 生命周期感知的协程作用域**，跟 MVVM 模式的关系是：**可以配合用，但不是因果关系**。

一个可能更好记的描述：

> ViewModel 不是 MVVM 的"VM"。它更像是 Android 框架提供的一个"记忆抽屉"——抽屉里的东西在屏幕旋转时不丢，在真正离开时清空。

### 行动清单

看完这篇文章，可以回头看一眼自己的项目：

- [ ] ViewModel 里有没有直接持有 Activity Context？
- [ ] 有哪些状态存在 ViewModel 但没有用 `SavedStateHandle`？进程死亡后能不能恢复？
- [ ] ViewModel 是不是太胖了？能不能抽一些逻辑到 UseCase 或 Repository？
- [ ] Compose 项目里 `collectAsState()` 有没有替换成 `collectAsStateWithLifecycle()`？

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**

