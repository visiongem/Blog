---
title: "车企 Android 面试复盘｜从 WebSocket 保活问到 AIDL ANR"
date: 2026-11-20T00:00:00+08:00
draft: false
cover:
  image: "https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/automotive-android-interview-cover.png"
  alt: "车企 Android 面试复盘封面"
isCJKLanguage: true
description: "整理车企 Android 面试的 9 个问题，用例子讲清 JNI、WebSocket 心跳与重连、AIDL 权限与 ANR、Activity 启动模式、Kotlin 协程、自定义 View、MQTT 和冷启动优化。"
categories: ["Android"]
tags: ["Android", "面试", "Kotlin", "网络", "性能优化"]
---

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/automotive-android-interview-cover.png)

## 1. JNI 是什么？Android 怎么调用 C/C++ 代码？

假设 Android App 想用一套现成的 C++ 图像处理库。界面用 Kotlin 写，处理图片的函数在 C++ 里。**JNI（Java 原生接口）让这两边可以在同一个 App 进程里互相调用。** [Android JNI 指南](https://developer.android.com/ndk/guides/jni-tips)

实际接入时，通常先把 C/C++ 代码编译成共享库（`.so` 文件），再用 `System.loadLibrary()` 加载它。Kotlin 用 `external` 声明要调用的方法，Java 用 `native`。NDK 是用来构建和调试这类原生代码的工具集。 [Android NDK 指南](https://developer.android.com/ndk/guides)

`JNIEnv*` 是原生代码访问 JNI 功能时用到的指针，每个线程要用自己的。C++ 自己创建的线程要调用 JNI，可以先用 `AttachCurrentThread()` 把当前线程登记到虚拟机，拿到自己的 `JNIEnv*`，在线程退出前再用 `DetachCurrentThread()` 解除登记。 [JNI 线程规则](https://developer.android.com/ndk/guides/jni-tips#threads)

## 2. 项目里的 WebSocket 怎么保活？

想象手机从 Wi-Fi 切到移动网络。这时旧连接可能已经断了，App 还没收到关闭通知。可以定期发一条很小的消息探测连接，这就是心跳。WebSocket 协议自带 `Ping` 和 `Pong`，正常情况下，一端发 `Ping`，另一端回 `Pong`。有的项目也会自己定义一套心跳消息和回复。超过设定时间还没收到回复，就把旧连接按失效处理。 [WebSocket Ping 和 Pong 规范](https://www.rfc-editor.org/rfc/rfc6455.html#section-5.5.2)

超时和切网都可能触发重连，最好统一交给一处代码管理，避免同时建两条新连接。连续失败时，可以把等待时间逐次翻倍，直到设定的上限，这叫指数退避。再加一点随机延迟，避免断网恢复时所有客户端一起重连。 [WebSocket 异常断线恢复规范](https://www.rfc-editor.org/rfc/rfc6455.html#section-7.2.3)

**连回去之后，还要看业务状态有没有恢复。** 比如订阅信息没保留，就重新订阅。如果服务端支持按序号补拉，客户端可以从最后一条已处理的消息往后补。项目里具体是谁发心跳、多久判超时，要按实际配置回答。如果谈到手机后台常驻，Android 的 Doze 省电模式还会限制网络访问。 [Android Doze 与应用待机](https://developer.android.com/training/monitoring-device-state/doze-standby)

![WebSocket 客户端发送 Ping，服务端回复 Pong。超时后安排重连，成功后按需恢复订阅或补消息。](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/automotive-android-interview-websocket-heartbeat.png)

## 3. AIDL 怎么用？跨应用调用怎么做权限控制？

假设界面进程要向另一个进程里的服务查询设备状态。这两个进程有各自的内存空间。**AIDL 用来约定双方能调用哪些方法、传什么参数，Binder 负责把调用送过去。** [Android AIDL 指南](https://developer.android.com/develop/background-work/services/aidl)

先把约定写进 `.aidl` 文件。编译会生成接收调用的 `Stub`，服务端实现它，并在 `onBind()` 里返回它。客户端用 `bindService()` 连接服务，连接成功后用 `Stub.asInterface()` 拿到接口。 [Android AIDL 指南](https://developer.android.com/develop/background-work/services/aidl)

跨进程时，客户端拿到的是代理 `Proxy`。它把参数装进 Android 可以传递的数据包 `Parcel`，由 Binder 送到服务端，再由 `Stub` 读出参数、执行方法。远程请求可能在多个线程上同时执行，如果它们会改同一份状态，服务端需要协调这些读写。 [Android AIDL 指南](https://developer.android.com/develop/background-work/services/aidl)

先管**谁能绑定服务**。只给本 App 用，就在 `AndroidManifest.xml` 里把服务设为 `android:exported="false"`。要开放给同签名的其他 App，可以设为 `android:exported="true"`，并用 `android:permission` 要求调用方持有自定义的 `signature` 级权限。调用方也要在自己的清单里用 `<uses-permission>` 申请该权限，Android 会检查签名。 [Service 清单属性](https://developer.android.com/guide/topics/manifest/service-element) · [Android 签名权限](https://developer.android.com/guide/topics/manifest/permission-element)

再管**绑定后能做什么**。比如同一个服务既能读设备状态，也能改配置，这两个方法可以有不同的授权规则。处理 Binder 调用时，用 `Binder.getCallingUid()` 拿到系统提供的调用方身份编号，再按方法做授权并检查参数。把耗时任务转交给工作线程前，先完成身份检查。 [Binder 调用方身份](https://developer.android.com/reference/android/os/Binder#getCallingUid()) · [Android 权限安全指南](https://developer.android.com/agents/skills/security/android-permissions-security/skill)

## 4. AIDL 为什么可能引发 ANR？

还是刚才的设备状态例子。假设点击按钮后，界面在主线程上调用远程 `getStatus()`。远程 AIDL 默认是同步调用，主线程要等服务返回才能往下走。如果服务正在慢慢读磁盘，主线程就可能一直等着，来不及处理点击和滑动。等得太久，系统就可能报 ANR，也就是应用无响应。 [Android ANR 排查指南](https://developer.android.com/topic/performance/anrs/diagnose-and-fix-anrs)

![同步 AIDL 调用的时间线。服务读取磁盘期间，App 主线程等结果，暂时无法处理新的点击。](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/automotive-android-interview-aidl-anr.png)

遇到这种情况，先看 ANR 日志里主线程停在哪里。如果停在 Binder 调用上，再用 Perfetto 看这次请求在服务端花了多长时间，是在等锁、等磁盘，还是卡在其他地方。需要同步结果的接口，可以把调用放到后台线程，同时缩短服务端处理时间。 [Android ANR 排查指南](https://developer.android.com/topic/performance/anrs/diagnose-and-fix-anrs)

如果可以晚一点拿结果，可以把接口拆成“提交查询”和“查询完成后通知客户端”。后一步就是回调。把提交查询声明为 `oneway` 后，跨进程调用方发出请求就可以继续执行，服务端稍后处理。同进程的本地调用依然同步。 [AIDL 的调用线程与 oneway](https://developer.android.com/develop/background-work/services/aidl)

## 5. Activity 有哪些启动模式？`taskAffinity` 管什么？

这里把一个 Activity 当成一个页面来讲，**task** 是用户完成一件事时打开的一组页面。假设你依次打开 `首页 A → 列表 B → 详情 C`，这个顺序就记在返回栈里。C 在栈顶，按返回可以回到下面的 B。下面用这组页面演示普通启动时的入栈和复用。 [Android 任务与返回栈](https://developer.android.com/guide/components/activities/tasks-and-back-stack)

- **`standard`** 是默认模式。从 C 再打开 B，会新建一个 B，栈变成 `A → B → C → B`。
- **`singleTop`** 只在目标页面已经处于目标栈顶部时复用。如果 C 配了 `singleTop`，再打开 C，就通过 `onNewIntent()` 把新请求交给当前的 C。如果 B 配了 `singleTop`，从 C 打开 B，仍会新建一个 B，因为旧的 B 在 C 下面。
- **`singleTask`** 可以复用已有实例。比如作为栈底的首页 A 配置了 `singleTask`，再次打开并复用 A 时，上面的 B、C 会被移除，A 收到 `onNewIntent()`。
- **`singleInstance`** 让这个 Activity 独占一个 task。它打开的其他页面会进入别的 task。
- **`singleInstancePerTask`** 要求这个 Activity 是所在 task 的第一个页面，每个 task 最多有一个实例。配合 `FLAG_ACTIVITY_NEW_DOCUMENT` 等特定启动参数，可以在不同 task 中各有一个实例。 [Activity 启动模式](https://developer.android.com/guide/topics/manifest/activity-element#lmode)

`taskAffinity` 描述一个页面倾向归属哪个 task。假设列表 B 用默认启动模式，即使 B 的 affinity 和 A 不同，从 A 普通启动 B 时，B 通常还是进 A 所在的 task。用 `FLAG_ACTIVITY_NEW_TASK` 启动 B 时，系统会考虑 affinity，也可能复用已有的 task。对于允许重新归属的页面，配置 `allowTaskReparenting="true"` 后，匹配的 task 回到前台时，页面还可能迁过去。 [Android 的 task affinity 规则](https://developer.android.com/guide/components/activities/tasks-and-back-stack#Affinities)

## 6. 怎么讲 Kotlin 协程？

点一下刷新，页面用 `viewModelScope.launch` 开始一次网络请求。当网络库用异步方式等待响应时，协程可以在等结果的地方暂停，主线程继续处理用户滑动。结果回来，协程再从暂停处接着执行。**挂起时，协程暂停，线程可以继续做别的事。** [Kotlin 协程基础](https://kotlinlang.org/docs/coroutines-basics.html)

`suspend` 表示函数可以挂起。它在哪个线程运行，要看协程的调度器。比如在主线程调用一个标了 `suspend` 的函数，函数里面直接同步读文件，主线程照样要等文件读完。负责读文件的代码可以用 `withContext(Dispatchers.IO)`，把这段工作交给适合处理文件读写的线程。 [Android 协程实践](https://developer.android.com/kotlin/coroutines/coroutines-best-practices)

追问 `launch` 和 `async` 时，可以这样答。`launch` 返回 `Job`，用来管理一个任务。`async` 返回 `Deferred`，可以用 `await()` 等结果。协程也要有明确的生命周期，例如 ViewModel 被清理时，`viewModelScope` 里的协程会被取消。取消协程后，已经发出的同步 Binder 调用仍可能继续等到返回。 [Kotlin 取消与超时](https://kotlinlang.org/docs/cancellation-and-timeouts.html) · [Android 的 ViewModelScope](https://developer.android.com/topic/libraries/architecture/coroutines#viewmodelscope)

## 7. 自定义 View 怎么讲？

用一个车速仪表盘举例。这个控件要决定自己有多大，再画出表盘和指针。

- **测量。** 如果控件需要自己决定宽高，可以重写 `onMeasure()`，根据父容器给的尺寸限制和控件期望的大小来计算。
- **准备绘制数据。** 宽高确定或发生变化后，可以在 `onSizeChanged()` 里算好表盘中心和半径。
- **绘制。** 在 `onDraw()` 里用 `Canvas` 画刻度和指针。像 `Paint` 这样会重复用的对象，可以提前创建。
- **摆放子控件。** 如果写的是容器控件 `ViewGroup`，还要在 `onLayout()` 里安排它的子 View 放在哪儿。 [Android 自定义绘制](https://developer.android.com/develop/ui/views/layout/custom-views/custom-drawing)

车速从 30 变成 40，控件大小照旧，只要更新角度并调用 `invalidate()` 请求重绘。如果表盘需要变大，控件希望占用的尺寸变了，就调用 `requestLayout()` 请求重新布局。 [Android View API](https://developer.android.com/reference/android/view/View)

如果是可以拖动指针的控件，还可能碰到滑动冲突。比如手势已经交给仪表盘，外层列表后来判断用户在上下滑，于是在 `onInterceptTouchEvent()` 里接管手势。这时仪表盘会收到 `ACTION_CANCEL`，可以结束当前的拖动状态。 [ViewGroup 的触摸事件分发](https://developer.android.com/develop/ui/views/touch-and-input/gestures/viewgroup)

## 8. 车云通信里的 MQTT 怎么理解？

例如，车端定期把电量发布到名为 `cars/42/battery` 的 **Topic**（主题）。云端服务订阅这个主题后，负责转发消息的 **Broker** 会把车端发来的电量转交给它。这就是发布订阅。WebSocket 提供双向通信，MQTT 规定消息怎么发布、订阅和交付，也可以通过 WebSocket 来传输。 [OASIS MQTT 5.0 规范](https://docs.oasis-open.org/mqtt/mqtt/v5.0/os/mqtt-v5.0-os.html)

**QoS** 指消息的交付级别。车端发给 Broker、Broker 再转给云端，是两段交付，各段可以用不同的 QoS。下面说的是其中一段的交付规则。

- **QoS 0** 最多交付一次，消息可能丢失。
- **QoS 1** 至少交付一次，接收方可能收到重复消息。
- **QoS 2** 通过额外的确认步骤实现恰好一次交付。如果业务代码主动把同一条指令发布两遍，协议会把它们当成两条消息，接收方仍需按指令 ID 做去重。 [MQTT 5.0 的 QoS 规则](https://docs.oasis-open.org/mqtt/mqtt/v5.0/os/mqtt-v5.0-os.html)

MQTT 自己也有保活机制。Keep Alive 设为非零时，如果客户端在约定时间内没有发送其他 MQTT 控制包，就需要发 `PINGREQ`。Broker 收到后回 `PINGRESP`。 [MQTT 5.0 的 Keep Alive 规则](https://docs.oasis-open.org/mqtt/mqtt/v5.0/os/mqtt-v5.0-os.html)

## 9. 启动优化除了 App Startup，还能做什么？

假设首页很快就显示了框架和加载动画，列表过一会儿才出来。**TTID** 量的是从启动到显示 App 的第一帧用了多久。**TTFD** 也从启动计时，直到应用报告首屏可用。比如列表已经显示、用户可以点开一项了，App 就可以调用 `reportFullyDrawn()` 告诉系统首屏准备好了。 [Android 应用启动指标](https://developer.android.com/topic/performance/issues/launch-time)

![启动时间线。TTID 从点开 App 到看到首帧，TTFD 从点开 App 到首屏内容可用。](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/automotive-android-interview-startup-ttid-ttfd.png)

App Startup 可以让接入的组件共用同一个启动入口（`ContentProvider`），并声明初始化顺序，例如 A 要用 B，就先初始化 B。自动初始化的工作仍会花掉启动时间。 [Jetpack App Startup](https://developer.android.com/topic/libraries/app-startup)

要定位启动慢在哪里，可以先用 Macrobenchmark 在同一设备和构建配置上反复测量冷启动。冷启动时，App 进程尚未运行，系统要先创建它。测出慢之后，再用 Perfetto 看启动期间主线程在忙什么。 [Macrobenchmark 启动指标](https://developer.android.com/topic/performance/benchmarking/macrobenchmark-metrics) · [Android 应用启动指南](https://developer.android.com/topic/performance/issues/launch-time)

找到耗时来源再改。如果首屏用不到某个 SDK，可以等第一次用时再初始化。如果主线程卡在读文件或处理大图，可以把合适的工作移到后台，并检查首页实际还要等多久。还可以评估 Baseline Profiles，让 Android 运行时提前编译常用的启动代码。改完用同样的方法再测一次，看看第一帧和首屏可用时间各缩短了多少。 [Android 启动优化指南](https://developer.android.com/topic/performance/issues/launch-time) · [Baseline Profiles 概览](https://developer.android.com/topic/performance/baselineprofiles/overview)

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**
