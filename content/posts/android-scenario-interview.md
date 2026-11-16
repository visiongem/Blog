---
title: "Android 场景面试 8 问：列表卡顿、ANR、内存与生命周期"
date: 2026-11-04T09:20:00+08:00
draft: false
aliases: ["/posts/position-list-realtime-performance-interview/"]
cover:
  image: "https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/ac0.png"
  alt: "Android 场景面试 8 问：性能、内存与生命周期"
  hiddenInSingle: true
isCJKLanguage: true
description: "围绕 Android 面经中的性能与交互考点，整理列表卡顿、Bitmap OOM、内存泄漏、ANR、启动优化、滑动冲突、DiffUtil 和 Flow 生命周期 8 道场景题，附参考回答与追问。"
categories: ["Android"]
tags: ["Android", "性能优化", "RecyclerView", "Flow", "面试"]
---

“RecyclerView 已经有复用了，为什么滑动还是卡？”“耗时操作都放到子线程了，为什么还会 ANR？”

这类问题需要把原理落到具体现场：先判断哪里出了问题，再解释怎么查、为什么这样改，以及如何验证。

**题目来源说明：** 下面的场景描述和追问经过重新编写，不是面试原话，也不代表某家公司的固定题库。公开面经只用于确认考点出现过，参考回答结合 Android 官方文档整理，来源见文末。

![Android 场景面试 8 问：性能、内存与生命周期](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/ac0.png)

## 1. 首页图文列表快速滑动时卡顿，你怎么定位和优化？

**场景：** 首页使用 RecyclerView 展示图文信息。高端机基本流畅，低端机快速滑动时明显掉帧。你接手后先做什么？

### 参考回答

我会先固定测试设备、数据量和滑动路径，抓取系统 trace，判断耗时发生在哪一段，而不是先调大缓存。

一次列表显示可能包含 View 创建、数据绑定、测量布局、绘制和图片处理。用 Perfetto 看主线程、RenderThread、线程调度及锁等待，再结合具体方法的 trace 定位：

| 现场 | 排查方向 |
|---|---|
| 新 Item 进入屏幕时卡 | 布局加载、View 创建、图片加载 |
| `onBindViewHolder()` 耗时长 | 同步 IO、复杂计算、重复格式化 |
| layout 耗时长 | 布局层级、重复测量、频繁 `requestLayout()` |
| 滑动中频繁 GC | 短生命周期对象、图片及中间结果的分配 |
| 数据刷新时才卡 | 全量通知、差分频率、变化动画 |

确认原因后分别处理：把 IO 和重计算移出绑定流程，图片按目标尺寸加载，简化测量布局，减少重复分配和不必要的刷新。优化布局要以测量结果为准，不能认为换成某一种容器就一定更快。

最后用同一套操作比较帧耗时分布、卡顿次数与内存变化。60Hz 的帧间隔约为 16.7ms，120Hz 约为 8.3ms，不能给所有设备套一个固定的业务计算预算。

### 追问：RecyclerView 已经复用 View，为什么还会卡？

复用主要降低创建 View 的成本。绑定、布局、绘制、图片处理仍然可能发生，上游数据计算也没有因此消失。已经复用的 View，如果每次绑定都做昂贵工作，同样会卡。

### 追问：调大缓存一定有用吗？

不一定。它可能减少部分创建或绑定，但会增加内存占用，也解决不了全量刷新和上游计算的问题。应在命中率、耗时与内存之间测量取舍。

参考：[Android 慢渲染诊断](https://developer.android.com/topic/performance/issues/render)、[Macrobenchmark 指标](https://developer.android.com/topic/performance/benchmarking/macrobenchmark-metrics)。

## 2. 一张图片文件只有几 MB，加载后为什么会 OOM？

**场景：** 用户从相册选择照片，进入预览页时偶尔 OOM。照片文件只有 3MB，同事认为“不可能是图片太大”。你怎么解释和处理？

### 参考回答

图片文件大小是压缩编码后的大小，不等于解码后的像素内存。以 4000 × 3000 的 ARGB_8888 图片为例，仅像素数据约占：

```text
4000 × 3000 × 4 字节 ≈ 45.8 MiB
```

如果同时保留原图、缩放结果和其他预览图，峰值会更高。

我会先读取图片尺寸，再按目标显示尺寸采样解码，避免完整解码后才缩小。普通列表或预览优先使用成熟图片库，并设置合适的目标尺寸、缓存预算及并发量。需要查看细节的超大图，可以考虑分块解码和按需加载。

排查时同时观察存活对象和瞬时分配峰值。如果内存主要消耗在多张正在处理的图片上，应控制并发；如果离开页面后图片仍被持有，则继续检查引用链。

### 追问：降低 JPEG 压缩质量能解决吗？

不一定。降低编码质量主要改变文件大小。在像素尺寸和解码格式相同的情况下，解码内存通常不会因此明显下降。需要降低解码尺寸，或根据画质、透明度要求选择合适的像素格式。

### 追问：OOM 就说明有内存泄漏吗？

不是。OOM 是分配失败，内存泄漏是已不需要的对象仍被持有。没有泄漏，也可能因单次分配过大或瞬时峰值过高而 OOM。

参考：[高效加载大图](https://developer.android.com/topic/performance/graphics/load-bitmap)。

## 3. 反复进入、退出页面后内存持续上涨，你怎么查？

**场景：** 一个详情页反复打开、关闭十几次后，内存明显上涨。页面没有崩溃，如何判断是否泄漏？

### 参考回答

内存上涨本身不能证明泄漏，缓存增长和暂时未回收也会出现这个现象。我会重复操作，检查已经销毁的 Activity 或 View 是否持续存活，用 LeakCanary 或堆转储找到从 GC Root 到这些对象的引用链。

重点检查几种所有权不匹配：

- 单例、静态集合保存了 Activity 或 View。
- 监听器注册到长期存活的对象，却没有注销。
- 消息队列中的 Runnable 捕获页面，且尚未执行或取消。
- 后台任务、动画或定时器继续持有已经退出的页面。
- Fragment 的 View 已销毁，但 Fragment 仍保存旧 View Binding。

修复要落在引用链上：由注册方负责注销，按任务用途取消工作，在 `onDestroyView()` 清理 View 引用。不能把引用全部换成弱引用就算解决。

修复后重复相同操作，确认旧页面能够回收，并检查正常返回和重新进入页面的功能。

### 追问：Handler 改成静态内部类就一定不泄漏了吗？

不一定。Message、Runnable 或其他成员仍可能持有 Activity。静态内部类只去掉一种隐式外部引用，不能代替完整的引用链分析和任务清理。

### 追问：为什么 Fragment 的 Binding 不等到 `onDestroy()` 再清？

Fragment 和它的 View 有不同的生命周期。Fragment 留在返回栈时，View 可能已经销毁；继续持有 Binding 会让旧 View 树无法及时释放。

参考：[Android 内存管理](https://developer.android.com/topic/performance/memory)、[Fragment 生命周期](https://developer.android.com/guide/fragments/lifecycle)、[LeakCanary 原理](https://square.github.io/leakcanary/fundamentals-how-leakcanary-works/)。

## 4. 耗时操作都放到子线程了，为什么还会 ANR？

**场景：** 团队已经把网络和数据库操作放到后台线程，线上仍然有 ANR。你会如何分析？

### 参考回答

我会先查看 ANR 类型、发生时间和线程现场。ANR 不只有“主线程直接执行耗时操作”这一种原因。

例如，后台线程拿着某把锁做磁盘操作，主线程随后也要获取这把锁，仍然会被阻塞。主线程调用 `Future.get()` 等待后台结果、进行缓慢的同步 Binder 调用，或者线程池过度并发造成 CPU 竞争，也可能影响响应。

对于组件执行超时，还要检查组件回调本身，以及它开始执行前主线程是否已经被其他工作挡住。

分析时，我会把主线程、锁持有线程、相关 Binder 调用和系统调度信息对起来。确认根因后再缩短锁范围、移除同步等待、控制后台并发，或优化组件启动路径。不同 ANR 类型及系统版本的超时条件存在差异，不应只背一个固定秒数来判断。

![后台线程持锁执行慢 IO，主线程等待同一把锁](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/ac1.png)

### 追问：主线程抓栈显示 `nativePollOnce`，是不是误报？

不能直接下结论。抓栈可能发生在阻塞已经结束之后，需要结合 ANR 原因、其他线程和时间线判断。不能因为这一刻主线程空闲，就排除之前的阻塞。

### 追问：再开一个线程处理能解决吗？

如果主线程仍然同步等待这个线程，问题并没有消失。需要让主线程能够继续处理事件，并在结果准备好后异步更新状态。

参考：[诊断和修复 ANR](https://developer.android.com/topic/performance/anrs/diagnose-and-fix-anrs)。

## 5. 接入多个 SDK 后冷启动变慢，你怎么优化？

**场景：** 接入统计、推送等 SDK 后，点击图标到首页可用的时间明显变长。直接把初始化全放到线程池可以吗？

### 参考回答

我会先明确测量的是冷启动、温启动还是热启动，再分别看首次显示时间 TTID 和完整显示时间 TTFD。首屏出现，不等于主要内容已经准备好。TTFD 需要应用在主要内容就绪且可用时通过 `reportFullyDrawn()` 或对应的 FullyDrawnReporter / Compose API 报告，系统不会自动知道这个时刻。

沿着启动链路检查 ContentProvider、Application、首个 Activity、布局加载和数据读取，找出首屏必须等待的工作。SDK 可能通过 ContentProvider 自动初始化，所以只检查 `Application.onCreate()` 不够。

然后按依赖关系拆分任务：首屏必须完成的留在关键路径；可延迟的按需初始化；能够后台执行的控制并发，并遵守 SDK 的线程要求。不要同时启动大量任务争抢 CPU 和磁盘，也不能破坏初始化顺序。

最后在相同设备和构建条件下重复测量，比较启动时间，同时检查首屏内容和交互是否真的更早可用。

### 追问：把任务放到 IdleHandler 就行了吗？

不行。IdleHandler 的回调仍在对应 Looper 的线程执行。主线程空闲时执行重任务，也会影响随后到来的输入和绘制；它不是后台线程。

### 追问：提前关闭启动页算不算优化？

只改变显示时机不够。如果首页仍无法操作、内容继续空白，用户的实际等待没有缩短。

参考：[应用启动时间](https://developer.android.com/topic/performance/issues/launch-time)、[MessageQueue.IdleHandler](https://developer.android.com/reference/android/os/MessageQueue.IdleHandler)。

## 6. 纵向列表嵌套横向轮播，滑动冲突怎么解决？

**场景：** 外层是纵向 RecyclerView，某个 Item 内是横向轮播。斜着滑时方向经常不符合预期，偶尔还影响点击。

### 参考回答

先确定交互规则：横向意图由轮播处理，纵向意图由外层列表处理。不要在 DOWN 时就提前决定所有权。

在 DOWN 记录起点，MOVE 超过 `touchSlop` 后，再结合横纵位移判断方向，避免把手指轻微抖动当作拖动。父容器通过 `onInterceptTouchEvent()` 决定是否拦截，子 View 在需要时调用 `requestDisallowInterceptTouchEvent()` 请求父容器不要拦截。该请求应在父容器接管之前发出；父容器已经拦截并向子 View 发送 CANCEL 后，不能靠这个调用把同一手势直接抢回来。

如果父容器中途接管事件，子 View 会收到 CANCEL，需要正确结束按压和拖动状态。还要检查点击、多指及边界滑动，不能只验证一次横滑。

对于父子共同消费滚动距离的需求，应使用嵌套滚动机制来协调；事件拦截解决谁接收触摸，嵌套滚动处理滚动距离如何协作，两者不能混为一谈。

### 追问：父容器在 DOWN 就返回 true 可以吗？

对这个场景通常不合适。方向尚未确定时就拦截，子控件将难以正常处理点击和横滑。

### 追问：横向内容已经滑到边缘，是否交给父容器？

先确认产品希望的手势行为。外层是纵向列表，不能仅因横向内容到边缘，就无条件把横向手势交出去；仍要考虑方向、控件能力和完整事件序列。

参考：[管理 ViewGroup 中的触摸事件](https://developer.android.com/develop/ui/views/touch-and-input/gestures/viewgroup)。

## 7. 列表每秒更新多次，已经用了 DiffUtil，为什么仍然卡？

**场景：** 一个行情列表有 1,500 条记录，价格持续变化。使用 ListAdapter 后，刷新时仍然卡顿，有时还闪烁。

### 参考回答

DiffUtil 用来找差异，不会替我们降低数据生成和绑定成本。如果每一行的展示内容都变了，差分后仍然会产生大量内容更新。后台比较也需要 CPU，提交太频繁仍然有代价。

我会分别测数据归并、展示计算、列表比较和绑定耗时。先按交易对共享行情计算，避免每条仓位重复做同样的工作；业务允许时合并展示更新，再控制提交列表的频率。必须完整处理的业务增量不能随意丢弃。

列表层面用稳定的业务 ID 判断实体，用影响 UI 的字段判断内容是否变化。价格或盈亏变化时通过 payload 局部绑定，避免刷新整行；检查变化动画、重复字符串格式化和布局请求是否造成额外开销。

提交的数据要保持不可变。没有变化的 Item 可以复用，变化的 Item 创建新对象，不必每次深拷贝所有内容。

### 追问：修改原列表，再 `submitList()` 为什么可能不刷新？

提交同一个列表实例可能直接被视为无需更新。即使复制了列表容器，如果新旧列表共享的可变 Item 已经被修改，旧内容基线也丢失了。需要新列表，并替换发生变化的 Item。

### 追问：有 payload 之后，可以不写完整绑定吗？

不可以。View 没有附着等情况下，payload 可能不会送达；完整绑定必须能够独立恢复正确界面，局部绑定只是优化。

### 追问：只算可见行就够了吗？

只能延后不影响全局业务的展示工作。如果总盈亏或排序依赖屏幕外的数据，相应计算仍然必须覆盖那些记录。

参考：[DiffUtil](https://developer.android.com/reference/androidx/recyclerview/widget/DiffUtil)、[RecyclerView.Adapter 的绑定约定](https://developer.android.com/reference/androidx/recyclerview/widget/RecyclerView.Adapter)。

## 8. 页面退出后，Flow 和定时刷新为什么还在执行？

**场景：** 从 Fragment A 进入 B，A 留在返回栈中但不再显示，日志里却仍有 A 的网络轮询和列表计算，该怎么管理？如果是按返回键把页面弹出返回栈，页面自身作用域的 ViewModel 通常会被清理；Activity 或导航图共享的 ViewModel 则取决于其所有者的生命周期。

### 参考回答

先分清 UI 收集、数据生产和共享连接分别由谁持有。停止收集不等于停止所有后台工作。

Fragment 的 UI 收集应绑定 View 的生命周期。下面的代码在 `onViewCreated()` 中注册一次，避免每次 `onStart()` 都重复启动外层协程：

```kotlin
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state ->
            render(state)
        }
    }
}
```

低于 STARTED 时取消这段收集，重新进入时再次启动。但如果轮询早已在 `viewModelScope` 或应用作用域中单独启动，它仍可能继续执行。

页面专属的轮询和展示计算可以由订阅需求驱动，例如使用 `SharingStarted.WhileSubscribed()` 控制共享上游，并在上游取消时释放监听或停止工作。默认在最后一个订阅者离开时立即停止；配置了 `stopTimeoutMillis` 则会延迟停止，其他订阅者仍存在时也不会停止。默认保留 replay 缓存，因此停止生产不等于清空旧状态。它不会自动关闭另一个作用域启动的 Socket 或 Service；CPU 密集循环也需要协作检查取消。

如果连接由多个页面共享，应按实际使用者管理订阅，不能一个页面退出就关闭全部连接。最后同时检查收集数量、定时任务、订阅数量和计算量，确认退出后确实释放了不再需要的资源。

![UI 停止收集与共享上游停止的生命周期边界](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/ac2.png)

### 追问：为什么有了 lifecycleScope 还需要 repeatOnLifecycle？

lifecycleScope 通常到对应 Lifecycle 销毁时才取消。页面停止显示但尚未销毁时，普通 `launch` 内的收集可能继续。`repeatOnLifecycle` 根据指定状态取消并重启代码块。

### 追问：STARTED 就一定是用户当前看到的页面吗？

不一定。具体取决于导航和容器的生命周期管理。若页面仍处于 STARTED 却不应刷新，还要结合实际页面可见需求管理任务。

参考：[生命周期感知的协程](https://developer.android.com/topic/libraries/architecture/coroutines)、[WhileSubscribed](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-sharing-started/-companion/-while-subscribed.html)。

## 面经出处与使用方式

以下公开记录用于说明考点来源，不用于证明本文的场景描述是原题；面经中的候选人回答也不作为技术结论的依据。

| 公开记录 | 本文对应考点 |
|---|---|
| [科大讯飞安卓客户端面经（作者主页可检索到该记录）](https://www.nowcoder.com/users/136981003) | 线程创建、内存泄漏、内存抖动、过度绘制、UI 卡顿 |
| [腾讯 QQ 社群安卓社招面经 7.23（话题页收录）](https://www.nowcoder.com/creation/subject/2ed342c87e3e480bbcb8f4849439a879) | RecyclerView、启动优化、内存泄漏监控 |
| [Android 系开发面经一面](https://www.nowcoder.com/feed/main/detail/4550a1ebbf884471983eff94b64fd068) | ANR、组件生命周期 |
| [Android 面经历史汇编](https://github.com/yinfork/Android-Interview/blob/master/Interview.md) | Bitmap 优化、OOM、滑动冲突、事件分发；属于二手历史汇编 |

练习时可以先不看答案，用一两分钟讲清楚：你要确认什么现象，用什么证据定位，为什么选择这个改法，以及如何证明问题已经解决。再用追问检查自己是否理解边界，而不是只记住工具名。

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**
