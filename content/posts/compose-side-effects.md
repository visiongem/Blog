---
title: "Jetpack Compose 副作用详解"
date: 2026-11-11T00:00:00+08:00
lastmod: 2026-11-12T00:00:00+08:00
draft: false
description: "用一个天气页面讲清 Compose 副作用：为什么需要它，每个 API 解决什么问题，以及怎么使用。先讲生活场景，再用简短代码说明各个 API 的用法。"
categories: ["Android"]
tags: ["Android", "Kotlin", "Jetpack Compose", "协程"]
---

你打开一个天气 App，屏幕上显示“北京，24°C”。这背后有两件事：

- App 去查询北京的天气。
- App 把查询到的结果显示出来。

**查询天气是一项要执行的操作，显示天气是在描述界面。** Compose 的副作用，主要就是在解决这些操作应该什么时候执行的问题。

下面沿着这个天气页面，逐个认识相关 API。每节先讲用途，再看代码；不熟悉代码的话，可以先读场景和解释。

![Compose 副作用：用一个天气页面讲清楚](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/ase0.png)

## 一、副作用到底是什么

先把“副作用”这个名字放下，想象一块商店里的价格牌。

价格牌写着“苹果 5 元一斤”。重新写一遍这句话，只是在更新展示内容。可如果你每写一遍，就顺便向顾客收一次钱，事情就变了。

收钱会影响价格牌之外的世界。即使后来把字擦掉，已经收走的钱也不会自动退回。

App 也是如此：

| 操作 | 产生了什么 |
|---|---|
| 根据已有温度，显示“24°C” | 一段界面内容 |
| 向服务器查询天气 | 一次网络请求 |
| 记录“用户打开了天气页” | 一条统计记录 |
| 开始监听温度传感器 | 一项持续运行的监听 |

后面这些超出“描述界面”本身的操作，就是这里说的副作用。

“副作用”不等于坏事。查询、保存、提醒，本来就是 App 正常工作的一部分。我们要做的是给这些操作安排合适的时机。

[官方文档](https://developer.android.com/develop/ui/compose/side-effects?hl=zh-cn)把 Side-effect 译为“附带效应”，本文统一称为“副作用”。

![显示天气与查询天气：描述界面和执行副作用的区别](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/ase1.png)

## 二、为什么不能把操作随手写进页面代码

Compose 是 Android 的一套界面开发工具。开发者告诉它“现在应该显示什么”，它负责更新界面。

天气从 24°C 变成 25°C，相关页面代码就可能重新执行，算出新的显示内容。Compose 把这种重新计算界面描述的过程叫作**重组**。

重新执行页面代码，不代表用户又打开了一次页面。

假设你把“记录一次访问”直接写在描述页面的代码里。温度变了，相关代码再执行一次，就可能多记录一次访问；其他状态更新，也可能继续增加次数。一次重组还可能中途被放弃，但已经发出的请求、已经写下的日志，不会跟着自动撤销。

因此，页面代码可以多次执行，而“查询一次”“记录一次”“注册一个监听器”要有自己的执行规则。副作用 API 就是用来写这些规则的。[Compose 编程思想](https://developer.android.com/develop/ui/compose/mental-model?hl=zh-cn)解释了重组为什么不能作为业务操作的触发依据。

先认识三个词就够了：

| 术语 | 在本文中怎么理解 |
|---|---|
| Composable | 用来描述界面的函数 |
| 组合 | Compose 当前维护的界面描述 |
| 重组 | 根据新的数据，重新执行相关界面代码 |

一块内容加入组合，叫“进入组合”；不再需要它、把它移除，叫“离开组合”。后文说“离开时取消”，指的是离开组合。仅仅把 App 切到后台，不一定会移除这块内容。

还有一个词叫 `State`：它是一份 Compose 能观察的数据。页面读取了它，之后它的值改变，Compose 就可以更新相关界面。例如页面正在读取温度，温度变化就可以带动文字更新。

## 三、LaunchedEffect 进入页面或条件变化时开始做事

打开北京的天气页，需要查北京天气。切换到上海，需要改查上海天气。

但如果只是页面上的按钮颜色变了，就没必要重新查询。

`LaunchedEffect` 适合给这类任务定规则。你告诉它要关注哪个条件，Compose 就在它进入组合时启动任务；之后，只有关注的条件变了，才重新启动。

这里关注的条件就是城市，代码里把这个条件叫作 **key**。

| 发生了什么 | 查询任务怎么处理 |
|---|---|
| 第一次进入页面，城市是北京 | 查询北京 |
| 页面重新执行，城市还是北京 | 保持原来的任务，不重新查 |
| 城市从北京改成上海 | 取消尚未完成的北京查询，开始上海查询 |
| 这段内容离开组合 | 取消尚未完成的任务 |

已经完成的任务，不会因为普通重组就再执行一次。取消也不会撤回服务器已经收到的请求。

### 代码示例：城市变化后查询天气

```kotlin
// 演示函数：用等待模拟联网，返回的是假数据。
// 实际项目中替换为自己的天气查询方法。
suspend fun queryWeather(city: String): String {
    delay(500)
    return "$city：24°C"
}

@Composable
fun WeatherPage(city: String) {
    var text by remember { mutableStateOf("正在查询") }

    LaunchedEffect(city) {
        text = "正在查询"
        text = queryWeather(city)
    }

    Text(text)
}
```

读这段代码时，先看三处：

- `remember`：让重组继续使用保存下来的状态。
- `LaunchedEffect(city)`：首次进入组合，或者城市变化时，运行里面的任务。
- `text = ...`：更新状态，让页面显示新的文字。

`LaunchedEffect` 会启动协程。这里可以把协程理解为“能够在等待时暂停、等结果来了再继续的任务”，例如等待网络返回。它不等于自动开启后台线程。

代码省略常规 Compose 和协程 import。后面的天气查询示例复用这个演示函数，真实请求的错误处理放在文末说明。

如果写成 `LaunchedEffect(Unit)`，`Unit` 是一个不会变化的 key。这表示在当前这次进入组合期间启动一次。离开后再进入，仍会重新启动，不能把它理解成“整个 App 永远只执行一次”。参见 [LaunchedEffect API](https://developer.android.com/reference/kotlin/androidx/compose/runtime/LaunchedEffect.composable)。

![LaunchedEffect 在进入组合、城市变化和离开组合时如何处理任务](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/ase2.png)

## 四、rememberCoroutineScope 用户点击后再做事

天气页还有一个“刷新”按钮。只有用户点击时，才需要查询。

这次的触发原因很明确：一次点击。可以直接在点击回调里安排操作。

如果操作需要等待，比如联网、播放滚动动画，就需要一个能启动协程的地方。`rememberCoroutineScope` 会提供这样的作用域。你可以把它理解成这块界面管理任务的地方：任务从这里启动，这块内容离开组合时，尚未完成的任务也会取消。

它本身不会开始刷新。真正开始做事的是点击时调用的 `launch`。

### 代码示例：点击按钮刷新天气

```kotlin
@Composable
fun RefreshWeatherButton(city: String) {
    val scope = rememberCoroutineScope()
    var text by remember { mutableStateOf("点击刷新") }
    var loading by remember { mutableStateOf(false) }

    Column {
        Button(
            enabled = !loading,
            onClick = {
                loading = true
                scope.launch {
                    try {
                        text = queryWeather(city)
                    } finally {
                        loading = false
                    }
                }
            }
        ) {
            Text("刷新")
        }
        Text(text)
    }
}
```

这里复用上一节的模拟查询。查询期间禁用按钮，避免反复点击。这个任务不会因为 `city` 参数变化而自动重启；如果需要“城市一变就取消旧查询”，应按上一节的思路用 `LaunchedEffect(city)` 管理。

不要把 `scope.launch` 随手放在 Composable 函数体里，那样页面代码每执行一次，就可能启动一次任务。放在 `onClick` 里，才表示点击时启动。

两者可以按触发原因区分：城市变了就查，用 `LaunchedEffect`；用户点了刷新再查，用 `rememberCoroutineScope`。如果点击只是执行一个普通的同步操作，比如跳转页面，直接在点击回调里调用就可以，不必为了使用副作用 API 再包一层。参见 [rememberCoroutineScope API](https://developer.android.com/reference/kotlin/androidx/compose/runtime/rememberCoroutineScope.composable)。

## 五、DisposableEffect 开始监听也要记得停止

天气页接上了一个室内温度传感器。打开这块内容后，App 开始接收温度；移除这块内容后，就应该停止接收。

这像订阅一个消息服务：订阅后会持续收到消息，不再需要时要取消订阅。

`DisposableEffect` 把开始和清理写在一起，避免只记得注册，却忘记注销。

### 代码示例：监听温度并在离开时注销

```kotlin
// 示例业务接口，不是 Compose 自带的 API。
// subscribe 注册监听，返回一个用于取消监听的函数。
interface TemperatureSource {
    fun subscribe(onTemperature: (Int) -> Unit): () -> Unit
}

@Composable
fun IndoorTemperature(source: TemperatureSource) {
    var temperature by remember(source) { mutableStateOf<Int?>(null) }

    DisposableEffect(source) {
        val unsubscribe = source.subscribe { newValue ->
            temperature = newValue
        }

        onDispose {
            unsubscribe()
        }
    }

    Text(temperature?.let { "$it°C" } ?: "等待温度")
}
```

`onDispose` 里写清理操作，并放在代码块末尾。这个例子注册了一个监听器，清理时就取消这一个监听器。

`source` 是 key。如果换了传感器，Compose 会先注销旧监听，再注册新监听。普通重组时，只要 `source` 没变，就不会反复注册。这里也用 `remember(source)` 保存温度，换传感器时先回到“等待温度”，避免继续显示旧传感器的读数。

需要成对处理的操作都可以考虑它，比如添加监听与移除监听、注册回调与注销回调。如果根本没有需要清理的事情，就不必专门使用 `DisposableEffect`。参见 [DisposableEffect API](https://developer.android.com/reference/kotlin/androidx/compose/runtime/DisposableEffect.composable)。

## 六、SideEffect 把界面里的设置同步给外部组件

天气页里有一张地图。页面选中了上海，地图也应该切到上海。

如果地图来自另一个库，不会自动读取 Compose 的状态，我们就需要调用地图提供的方法，把城市告诉它。

`SideEffect` 适合做这种同步：等 Compose 确认这次组合成功完成后，再更新外部对象，避免把一次尚未成功应用的界面更新提前通知出去。

### 代码示例：同步地图当前城市

```kotlin
// 示例业务接口，表示外部地图组件提供的能力。
interface WeatherMap {
    fun setCity(city: String)
}

@Composable
fun SyncMapCity(map: WeatherMap, city: String) {
    SideEffect {
        map.setCity(city)
    }
}
```

这个无 key 的版本，在执行到它的每次成功组合后都会运行，首次组合也包括在内。如果当前组件被跳过，或者这次重组被放弃，就不会因为这次重组执行它。

这里适合执行简短的同步操作，例如更新一个设置。`SideEffect` 不会启动协程，里面不能直接调用挂起函数，也不适合放会长时间阻塞的工作。

“组合成功”不代表屏幕已经完成绘制。需要读取布局尺寸时，要用相应的布局 API。

因为它可能多次执行，所以适合重复设置也没问题的操作。“把城市设为上海”，多设一次仍然是上海；“把访问次数加一”，多做一次就多了一次计数。后者需要另行安排触发条件。参见 [SideEffect API](https://developer.android.com/reference/kotlin/androidx/compose/runtime/SideEffect.composable)。

## 七、rememberUpdatedState 事情继续做但使用最新的信息

天气页有一条提示，显示 5 秒后结束。结束时执行什么操作，由外层页面传入。

等到第 2 秒时，外层页面传来了新的后续操作。我们希望原来的 5 秒倒计时继续，到时执行新的操作，而不是从第 2 秒起重新等 5 秒。

`rememberUpdatedState` 就是为这种情况准备的：**不重启原任务，但让它在需要时拿到最新值。**

| 时间 | 应该发生什么 |
|---|---|
| 第 0 秒 | 开始等待 5 秒 |
| 第 2 秒 | 更新结束时要执行的操作，继续等待 |
| 第 5 秒 | 调用最新的操作 |

### 代码示例：倒计时继续但更新结束回调

```kotlin
@Composable
fun WeatherTip(onFinished: () -> Unit) {
    val latestOnFinished by rememberUpdatedState(onFinished)

    LaunchedEffect(Unit) {
        delay(5_000)
        latestOnFinished()
    }

    Text("今日有雨，出门记得带伞")
}
```

`onFinished` 就是“结束时要执行的操作”，开发中把这种传入的函数称为回调。示例只负责到时调用它；如果要隐藏提示，由外层在这个回调里更新显示状态。

`rememberUpdatedState` 在重组时复用同一个 State 对象，并把里面保存的回调更新为这次传入的值。任务在 5 秒后读取它，就能拿到最新回调。不要在等待之前先把回调取出并保存到普通变量，否则之后调用的仍然是旧值。

如果把 `onFinished` 放进 `LaunchedEffect` 的 key，它变化时就会重启倒计时；如果直接在原任务里调用最初捕获的 `onFinished`，则可能调用旧回调。

这个例子假设当前是一条持续存在的提示。如果提示本身换成了另一条，并且需要重新计时，就应把提示 ID 作为 key。

key 和 `rememberUpdatedState` 解决的是两种需求：城市变了，原查询需要换成新查询，用 key；任务应该继续，只想更新它稍后使用的信息，用 `rememberUpdatedState`。参见 [rememberUpdatedState API](https://developer.android.com/reference/kotlin/androidx/compose/runtime/rememberUpdatedState.composable)。

## 八、produceState 查询完以后把结果交给页面

查询天气时，页面先显示“正在查询”，拿到结果后再显示“北京：24°C”。

这里要做两件事：运行查询任务，把查询结果放进页面能观察的数据里。

`produceState` 把这两件事放在一起。它返回一个 State，任务通过更新这个 State，让页面拿到新结果。

### 代码示例：从查询结果得到页面状态

```kotlin
@Composable
fun WeatherResult(city: String) {
    val text by produceState(
        initialValue = "正在查询",
        key1 = city
    ) {
        value = "正在查询"
        value = queryWeather(city)
    }

    Text(text)
}
```

这里仍使用前面的模拟查询函数。

- `initialValue` 是这份状态刚创建时的值。
- `city` 是 key，城市变化就取消旧任务、运行新任务。
- `value = ...` 是把结果写入返回的 State。

key 变化不会自动把旧状态重置成 `initialValue`，所以示例在每次查询开始时主动写入“正在查询”。新任务执行前，仍可能短暂读到旧结果；对这个过渡有严格要求时，可以让结果携带城市并校验后再展示。

第三节的 `LaunchedEffect` 也能完成这件事，只是需要自己准备 State，再在任务里更新。`produceState` 适合把“运行任务并提供状态”封装起来。它内部也使用了 `remember` 和 `LaunchedEffect`。参见 [produceState 源码](https://github.com/androidx/androidx/blob/androidx-main/compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/ProduceState.kt)。

## 九、derivedStateOf 只关心一个判断结果有没有变

天气页有一长串未来预报。用户向下滚动后，页面显示“回到顶部”按钮。

滚动位置一直在变，但按钮只关心一件事：有没有离开列表第一项。

| 第一个可见项目的索引 | 是否显示按钮 |
|---|---|
| 0 | 不显示 |
| 1 | 显示 |
| 2 | 显示 |
| 3 | 显示 |

从 1 滚到 2，再滚到 3，按钮都应该显示。位置变了，但这个判断的答案没变。

`derivedStateOf` 可以让页面读取这个判断结果，减少仅仅因为位置变化而产生的不必要重组。

### 代码示例：根据滚动位置显示按钮

```kotlin
@Composable
fun BackToTopButton(listState: LazyListState) {
    val scope = rememberCoroutineScope()

    val showButton by remember(listState) {
        derivedStateOf {
            listState.firstVisibleItemIndex > 0
        }
    }

    if (showButton) {
        Button(onClick = {
            scope.launch { listState.animateScrollToItem(0) }
        }) {
            Text("回到顶部")
        }
    }
}
```

`listState` 由外层列表传入。`firstVisibleItemIndex` 表示当前第一个可见项目的索引，索引从 0 开始。

这个组件读取的是 `showButton`。索引虽然经常变化，但判断结果相同时，可以避免由这条状态依赖引起的重组。其他参数变化仍可能导致重组。

`derivedStateOf` 会追踪计算中读取的 Compose 状态，本身也有开销。普通的姓名拼接、简单计算，直接算通常就够了，不必一律套上它。

它本身只是计算状态，不会发请求，也不会启动任务。计算块里应该只计算结果。参见 [derivedStateOf 源码](https://github.com/androidx/androidx/blob/androidx-main/compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/DerivedState.kt)。

## 十、snapshotFlow 观察变化然后做一件事

还是同一个预报列表。这次不需要显示按钮，而是想在用户第一次向下阅读时记录一条日志。

我们需要观察滚动状态，并在满足条件时执行操作。

`snapshotFlow` 把 Compose 状态的变化交给 Flow 处理。Flow 可以理解为会陆续送来结果的数据流；代码开始接收这些结果，叫作“收集”。

这个例子的处理过程是：

**观察滚动位置 → 判断是否离开第一项 → 第一次满足时记录日志。**

### 代码示例：第一次向下阅读时记录日志

```kotlin
@Composable
fun TrackReading(listState: LazyListState) {
    LaunchedEffect(listState) {
        snapshotFlow { listState.firstVisibleItemIndex > 0 }
            .filter { hasScrolled -> hasScrolled }
            .take(1)
            .collect {
                android.util.Log.d("Weather", "用户向下阅读了预报")
            }
    }
}
```

这几步可以逐行读：

- `snapshotFlow`：观察其中读取的 Compose 状态。
- `filter`：只保留条件为真的结果。
- `take(1)`：拿到第一个满足条件的结果后结束。
- `collect`：收到结果时执行这里的操作。

开始收集时，它会先发出当前的判断结果。因此，如果开始时就已滚过第一项，也会记录日志。这里的“一次”只针对当前这次收集；这个调用离开组合后再进入，可以再次记录。

Flow 本身不会替你选择何时开始收集，所以这里用 `LaunchedEffect` 管理收集任务。没有开始收集时，`snapshotFlow` 就不会开始观察。

`derivedStateOf` 适合给页面提供一个计算结果，例如“是否显示按钮”；`snapshotFlow` 适合观察状态变化，再交给后续代码处理，例如过滤后记录日志。

观察块里只读状态，实际操作放在 `collect` 中。它会跳过与上次结果相等的值，也不保证捕获每个快速变化的中间值，不能把它当作逐条保存事件的记录本。参见 [snapshotFlow 源码](https://github.com/androidx/androidx/blob/androidx-main/compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/SnapshotFlow.kt)。

## 十一、按你要做的事来选

先想清楚需求，再查名字，比从八个英文名开始背更容易。

| 你要做什么 | 用哪个 API |
|---|---|
| 页面进入组合就查询，城市变了重新查 | `LaunchedEffect` |
| 用户点击后，再启动一个需要等待的任务 | `rememberCoroutineScope` |
| 开始监听，移除内容时停止监听 | `DisposableEffect` |
| 组合成功后，把当前设置同步给外部组件 | `SideEffect` |
| 倒计时继续，但结束时要用最新的操作 | `rememberUpdatedState` |
| 运行查询，把结果作为状态交给页面 | `produceState` |
| 滚动位置经常变，只关心按钮该不该显示 | `derivedStateOf` |
| 观察滚动变化，满足条件后记录日志 | `snapshotFlow` |

这八个 API 都出现在官方副作用指南中，但各自负责的事情不同。有的运行任务，有的管理任务，有的计算或转换状态。它们可以一起使用，不必强行选一个包办所有事情。

![根据天气页面中的八种需求选择 Compose API](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/ase3.png)

## 十二、写实际项目时再看这些细节

前面已经介绍了官方指南中的八个 API。下面补充实际开发中的边界条件和其他相关 API，第一次阅读可以先跳过，遇到对应问题时再回来查。

### key、remember 和取消任务的几个细节

key 表示哪些参数变化后需要重新开始。多个参数都影响任务时，可以一起传入，例如 `LaunchedEffect(city, repository)`。

Compose 在重组执行到这个调用时，比较前后的 key。修改普通对象内部的字段，不等于换了 key，也不一定会触发重组。优先使用城市名、用户 ID 等能明确表达任务条件的值。

只在 `LaunchedEffect` 的任务里读取 State，不会让任务随该 State 的每次变化自动重启。需要重启就把相应值作为 key；需要持续观察，可以用 `snapshotFlow`。

`remember` 用来记住对象或计算结果。它的计算块仍在组合过程中执行，所以不能用 `remember { 发请求 }` 替代 effect。`rememberSaveable` 可以帮助保存可恢复的状态，同样不是执行副作用的工具。

列表的 `items(key = ...)` 用来识别是哪一个列表项；effect 的 key 用来决定任务是否重启。列表重排时，仅设置 effect 的 key，不能替代稳定的列表项 key。参见 [Composable 生命周期](https://developer.android.com/develop/ui/compose/lifecycle)。

在 Android UI 中，`LaunchedEffect` 通常在主线程开始执行。挂起函数不等于自动切换到后台线程，真实的数据层应保证可以从主线程安全调用。

协程取消需要任务配合。如果捕获 `Exception`，要先把 `CancellationException` 重新抛出，不能当作普通失败吞掉。真实查询还需要处理网络错误；前面的演示函数只返回假数据，没有这些失败分支。参见 [Android 协程最佳实践](https://developer.android.com/kotlin/coroutines/coroutines-best-practices)。

### produceState 接收持续推送的数据时怎么清理

`produceState` 也能接入回调。比如复用前面的 `TemperatureSource`，把传感器持续报告的温度变成 State：

```kotlin
@Composable
fun rememberTemperature(source: TemperatureSource): State<Int?> =
    produceState<Int?>(initialValue = null, key1 = source) {
        value = null
        val unsubscribe = source.subscribe { temperature ->
            value = temperature
        }

        awaitDispose {
            unsubscribe()
        }
    }
```

`awaitDispose` 会让协程在这里等待；当 key 变化或调用离开组合，导致协程取消时，它执行清理代码。数据源应保证注销生效后不再发送旧回调。

`produceState` 返回的状态使用默认的相等判断，重复写入相等值，不会因此触发重组。快速写入多个值，页面也可能只读到最新值。State 表达的是当前情况，不保证把每次变化都逐条交付。

这些行为见 [produceState 源码](https://github.com/androidx/androidx/blob/androidx-main/compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/ProduceState.kt)。

### 已有 Flow 时怎么显示，以及 snapshotFlow 如何去重

如果 ViewModel 已提供 `StateFlow`，Android 页面通常直接使用 `collectAsStateWithLifecycle`：

```kotlin
val weather by viewModel.weather.collectAsStateWithLifecycle()
```

它来自 `androidx.lifecycle.compose`，默认在当前 Lifecycle 至少为 `STARTED` 时收集，低于该状态时停止。普通 Flow 需要提供 `initialValue`；StateFlow 自带当前值。普通 `collectAsState` 则不会自动加入这个生命周期条件。

停止收集只表示当前订阅停止了。上游热流是否继续运行，要看它自己的作用域和共享策略。LiveData 的 `observeAsState`、Rx 的 `subscribeAsState` 也属于把外部数据变成 Compose State 的适配工具。参见 [Compose 支持的状态类型](https://developer.android.com/develop/ui/compose/state#other-supported-types-of-state)。

反过来，`snapshotFlow` 把 Compose 状态变成 Flow。它比较的是读取块的返回值：连续的 `true` 不会重复发出，但观察到 `false` 之后，再观察到 `true`，就可以再次发出。

如果先观察索引，再用 `map` 转成 Boolean，多个不同索引可能都变成 `true`，这时可在 `map` 后使用 `distinctUntilChanged()`。读取块只能读取状态，不应修改状态或执行上报；它也无法自动观察普通 Kotlin 变量。参见 [snapshotFlow 源码](https://github.com/androidx/androidx/blob/androidx-main/compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/SnapshotFlow.kt)。

### 切到后台时也要停止工作，用哪些 Lifecycle API

页面暂时看不见，不等于内容已经离开组合。例如按 Home 键回桌面，组合可能仍然保留。因此不能只靠 `LaunchedEffect` 或 `DisposableEffect` 判断前后台。

Lifecycle 表示当前页面或宿主处于什么生命周期状态。`LocalLifecycleOwner` 提供当前的 LifecycleOwner，它可能属于导航页面，不一定是整个 Activity。

| API | 什么时候工作 | 什么时候清理 |
|---|---|---|
| `LifecycleEventEffect` | 收到指定生命周期事件时执行操作 | 不提供配对清理 |
| `LifecycleStartEffect` | 收到 `ON_START` 时开始 | `ON_STOP` 或离开组合时 |
| `LifecycleResumeEffect` | 收到 `ON_RESUME` 时开始 | `ON_PAUSE` 或离开组合时 |

例如，页面可见期间才监听温度：

```kotlin
LifecycleStartEffect(source) {
    val unsubscribe = source.subscribe { temperature = it }

    onStopOrDispose {
        unsubscribe()
    }
}
```

这里复用前面的传感器接口和温度状态。key 变了，会清理旧监听并重新注册生命周期观察者。如果此时尚未达到 `STARTED`，需要等到下一次 `ON_START` 才开始监听。

另两种 API 的形式是：

```kotlin
LifecycleEventEffect(Lifecycle.Event.ON_RESUME) {
    // 收到事件时执行一次同步操作
}

LifecycleResumeEffect(controller) {
    controller.resume() // 示例业务对象提供的方法

    onPauseOrDispose {
        controller.pause()
    }
}
```

`LifecycleStartEffect` 必须在末尾提供 `onStopOrDispose`，`LifecycleResumeEffect` 必须提供 `onPauseOrDispose`。其中不能直接调用挂起函数。注册观察者时，Lifecycle 可能立即发送用于同步当前状态的事件；`LifecycleEventEffect` 不支持监听 `ON_DESTROY`。

这些 API 来自 `androidx.lifecycle:lifecycle-runtime-compose`，从 Lifecycle 2.7.0 起提供。参见 [Compose 中的 Lifecycle](https://developer.android.com/topic/libraries/architecture/lifecycle)。

如果要在可见期间持续收集 Flow，可以写成：

```kotlin
LaunchedEffect(owner, messages) {
    owner.lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
        messages.collect { message ->
            // 处理消息
        }
    }
}
```

其中 `owner` 来自 `LocalLifecycleOwner.current`，`messages` 是业务层传入的 Flow。达到 `STARTED` 时开始收集，低于它时取消这次收集，再次达到时重新收集。停止期间的消息是否丢失或重放，取决于上游 Flow 的设计。参见 [repeatOnLifecycle 源码](https://github.com/androidx/androidx/blob/androidx-main/lifecycle/lifecycle-runtime/src/commonMain/kotlin/androidx/lifecycle/RepeatOnLifecycle.kt)。

### RetainedEffect 怎样让资源跨屏幕旋转保留

旋转屏幕等配置变化，可能让旧组合销毁、新组合重新建立。如果某项资源不应该随之重建，可以考虑 `retain` 与 `RetainedEffect`。

`retain` 保存对象；`RetainedEffect` 管理与它一起保留的操作，在不再保留时执行 `onRetire`：

```kotlin
// 示意片段：factory 创建轻量的业务连接对象。
// factory 与 connection 都不能持有旧 Activity 或 View。
val connection = retain(city, factory) {
    factory.create(city)
}

RetainedEffect(connection) {
    connection.start()

    onRetire {
        connection.close()
    }
}
```

这些是业务方法，不是 Compose 提供的网络连接 API。真正的外部连接在 `start` 中建立。

`RetainedEffect` 来自 `androidx.compose.runtime:runtime-retain` 的 1.10 系列，Android 跨配置变更的保留支持由 Compose UI 1.10.0 起提供。它能否保留，取决于当前 `RetainedValuesStore` 和宿主的配置。

如果宿主允许保留暂时退出的内容，恢复时同一个调用和 key 匹配，就不用重新初始化；如果宿主不保留，它也会在离开组合时清理。key 改变时，如果旧内容仍在保留期，旧资源的清理可能延后，不能假定新旧资源绝不重叠。

它不负责前后台暂停。资源要保留多久，与什么时候允许它工作，是两件需要分别安排的事。参见 [状态保留范围](https://developer.android.com/develop/ui/compose/state-lifespans)和 [RetainedEffect API](https://developer.android.com/reference/kotlin/androidx/compose/runtime/retain/RetainedEffect.composable)。

### 新版 SideEffect 也支持 key，以及业务任务该放在哪里

正文介绍的是官方副作用指南中的无 key 用法。Compose Runtime 1.12.0 起还提供带 key 的 `SideEffect`：

```kotlin
SideEffect(map, city) {
    map.setCity(city)
}
```

首次成功进入组合时执行；之后只有 key 改变才再次执行。它仍然不启动协程，也不提供清理回调。使用前确认项目版本，参见 [SideEffect API](https://developer.android.com/reference/kotlin/androidx/compose/runtime/SideEffect.composable)。

真实项目中，需要跨配置变更继续运行的业务加载，通常交给 ViewModel；页面中的提示、滚动、动画，则适合跟随组合。需要在进程结束后仍可靠安排的工作，再考虑 WorkManager 等机制。

如果某个方法内部使用 `viewModelScope` 启动任务，那么任务就属于 ViewModel 的作用域。即使在 `LaunchedEffect` 里调用这个方法，也不会自动把任务改成跟随组合。

最后，不要让副作用不断改变自己的触发条件。例如 `count` 是 Compose 状态，`LaunchedEffect(count)` 里面又执行 `count++`，就会让任务反复重启。计算“按钮是否可点”这样的简单判断，也直接由当前输入计算即可，不必再建一份 State、通过副作用同步过去。

## 写在最后

回到开头的天气页面，显示“24°C”、点击刷新、持续接收温度，看起来都发生在同一个页面里，但它们需要不同的执行规则。

页面可以反复计算，查询什么时候重来、监听什么时候停止，都要单独安排。Compose 的这些 API，就是帮我们把规则写清楚。

刚开始不必记住所有名字。先分清“条件变化就做”“用户点击才做”“开始之后要清理”，就能找到最常用的 `LaunchedEffect`、`rememberCoroutineScope` 和 `DisposableEffect`。需要更新回调、把结果交给页面、观察状态时，再找其他工具。

下次写代码前，可以先用一句普通话描述需求：“换城市就重新查询，换按钮颜色不用查。”或者：“开始接收温度后，这块内容移除时要停止接收。”需求写清楚了，选哪个 API 就有了依据。

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**