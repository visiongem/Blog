---
title: "Android 开发面经 12 问"
date: 2026-10-18T18:00:00+08:00
draft: false
categories: ["Android"]
tags: ["Android", "面试", "网络", "性能优化"]
---

这 12 道题来自某书网友分享的一场 Android 一面。

下面是我用 Codex 对原题进行答案梳理，顺便说一句，感觉 Codex 这种 AI 梳理出来真是一板一眼的，还不如我直接把题目丢给豆包，豆包的回答呢～哈哈。

当然下面的回答我已经让 Codex 按照豆包回答整理过了，凑合着背吧。

背面试题关键还在于理解，理解后用自己话讲出来记忆才深刻，不然就我们这大龄程序员，哪里记得下那么多八股文。

![Android 开发面经 12 问封面](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/interview12_0.png)

## 1. 子线程为什么不能直接更新UI？有什么办法解决子线程更新UI的问题？

### 原理梳理

#### 为什么不能直接更新 UI

核心原因是 Android 的 UI 控件不是线程安全的。

比如主线程正在根据 TextView 的文字计算宽度，子线程却把文字改了，测量和绘制就可能用到不同的数据。View 没有通过一套同步锁来保证这些操作可以并发执行，所以 Android 要求我们在主线程操作 UI。

框架也做了线程检查。例如，布局请求传到 `ViewRootImpl.requestLayout()` 时，会调用 `checkThread()`，检查当前线程是不是创建这个 ViewRootImpl 的线程。普通 Activity 页面中就是主线程。检查不通过，就抛出 `CalledFromWrongThreadException`。具体可以看 [AOSP 的 ViewRootImpl 实现](https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/view/ViewRootImpl.java)。

有时子线程改 UI 没崩溃，是因为操作没有走到线程检查，例如页面还没有连接到 ViewRootImpl。

偶然的没有报错不代表线程安全，也不表示就能这么去做，报错不可预期，规范依然是禁止在子线程更新 UI。

#### 子线程拿到结果后，怎么更新 UI

常用的四种方法，目的都是让 UI 代码在主线程执行：

1. `View.post {}`：通过 View 投递任务。View 已经 attach 时，会使用它关联的 Handler；还没有 attach 时，任务通常先保存，等 attach 后再执行。
2. `Activity.runOnUiThread {}`：当前就在主线程则直接执行，否则通过 Handler 投递到主线程。
3. **主线程 Handler**：用 `Handler(Looper.getMainLooper()).post {}` 指定任务在主线程执行。页面销毁时取消不再需要的回调，避免延迟任务一直持有页面。
4. **Kotlin 协程**：耗时工作放到合适的调度器，结果回来后在 `Dispatchers.Main` 更新界面。协程要绑定生命周期。

Fragment 中加载数据的例子：

```kotlin
viewLifecycleOwner.lifecycleScope.launch {
    val result = withContext(Dispatchers.IO) {
        repository.loadData() // 阻塞式 I/O 放到后台线程
    }
    binding.textView.text = result // 返回主线程
}
```

View 销毁后，这个作用域会取消。也可以由 ViewModel 加载数据、页面收集状态，但 ViewModel 不应持有 TextView 或 ViewBinding。

### 面试口述

先记住：**View 不支持多线程同时改，子线程拿到结果后，把 UI 更新交回主线程。**

> 因为 Android 的 UI 控件不是线程安全的，主线程绘制时，子线程同时修改属性，就可能出现界面状态错乱。Android 会在 ViewRootImpl 中做线程检查，触发检查时，如果不是创建它的线程，就会抛出 CalledFromWrongThreadException。
>
> 解决办法都是让主线程执行 UI 更新，常用的有 View.post、Activity.runOnUiThread、主线程 Handler，以及协程的 Dispatchers.Main。耗时任务在后台做，结果回到主线程显示，同时注意页面销毁后取消不再需要的任务。

## 2. Java/Kotlin中创建线程池有哪些正确方式？线程池有哪几个核心参数？

### 原理梳理

#### 创建线程池时，先把资源上限定下来

Java 和 Kotlin/JVM 都可以使用 `ThreadPoolExecutor`，明确设置线程数、队列容量和拒绝策略。

面试中常说“不推荐直接使用 Executors 创建线程池”，主要担心这些默认配置：

- `newFixedThreadPool`、`newSingleThreadExecutor` 使用无界任务队列。提交速度长期超过处理速度，任务就会积压，增加内存占用。
- `newCachedThreadPool` 使用 `SynchronousQueue`，最大线程数设置为 `Integer.MAX_VALUE`。并发任务多且长时间不结束时，可能创建过多线程。

因此，业务负载不可控时，应显式配置资源上限。“禁止使用 Executors”通常是团队规范，并非 Java 限制。参见 [Executors 官方说明](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/Executors.html)。

下面仅演示配置方式，数值需按任务耗时和并发量调整：

```kotlin
val executor = ThreadPoolExecutor(
    2,                                 // 核心线程数
    5,                                 // 最大线程数
    10L,                               // 空闲超时时间
    TimeUnit.SECONDS,                  // 时间单位
    LinkedBlockingQueue<Runnable>(10),  // 明确队列容量
    Executors.defaultThreadFactory(),   // 线程工厂
    ThreadPoolExecutor.AbortPolicy()    // 拒绝策略
)
```

线程池应复用，按所属模块的生命周期关闭，并处理提交失败。

#### 七个参数分别管什么

1. **`corePoolSize`：核心线程数。** 默认按需创建，空闲后也保留。可提前启动，也可通过 `allowCoreThreadTimeOut(true)` 允许空闲回收，此时超时时间必须大于 0。
2. **`maximumPoolSize`：最大线程数。** 限制工作线程总数，但它是否生效还取决于队列能不能继续接收任务。
3. **`keepAliveTime`：空闲超时时间。** 默认控制超出核心数量的空闲线程多久退出。
4. **`unit`：时间单位。** 例如秒或毫秒。
5. **`workQueue`：等待任务的阻塞队列。** `ArrayBlockingQueue` 有界；`LinkedBlockingQueue` 不指定容量时接近无界；`SynchronousQueue` 不存储任务，要直接交给接收线程。
6. **`threadFactory`：线程工厂。** 负责创建线程，常用来设置线程名，方便定位问题。
7. **`handler`：拒绝策略。** 线程池关闭，或者线程和队列都无法接收新任务时触发。

部分构造函数重载会提供默认线程工厂和拒绝策略。参见 [ThreadPoolExecutor 官方说明](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadPoolExecutor.html)。

常见拒绝策略也要知道后果：`AbortPolicy` 抛异常；`CallerRunsPolicy` 由提交者执行，但线程池关闭后会丢弃；`DiscardPolicy` 直接丢弃；`DiscardOldestPolicy` 尝试丢掉队头任务，再提交新任务。Android 中尤其要小心 CallerRunsPolicy，提交者如果是主线程，耗时工作就会卡住界面。

#### 任务提交以后，按什么顺序处理

1. 工作线程数小于核心线程数，创建线程执行新任务。
2. 已达到核心线程数，先尝试入队。
3. 队列放不下，再尝试创建线程，直到最大线程数。
4. 线程也不能再创建，执行拒绝策略。

按上面的配置，假设任务一直未结束：前 2 个执行，接下来 10 个排队，再来 3 个触发扩容，之后的新任务被拒绝。

#### 定时任务和协程怎么选

定时任务可以用 `ScheduledThreadPoolExecutor`，但直接调用它的构造器也不会消除无界延迟队列的风险。仍要控制提交量，并按需要设置 `removeOnCancelPolicy`，及时移除已取消的任务。[定时线程池官方说明](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledThreadPoolExecutor.html)。

Kotlin 业务通常用协程管理并发和取消。`Dispatchers.IO` 适合阻塞式 I/O，在 JVM 实现中与 Default 共享线程资源，不能简单理解为某个独立的 ThreadPoolExecutor。CPU 密集计算使用 Default。[协程调度器官方说明](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-i-o.html)。

### 面试口述

先记住：**线程数要有限，队列要有限，满了要有处理办法。执行顺序是核心线程 → 队列 → 扩容 → 拒绝。**

> 创建线程池，我一般直接用 ThreadPoolExecutor，把线程数量、队列容量和拒绝策略设置清楚。因为 Executors 的固定线程池默认使用无界队列，任务可能一直堆积；缓存线程池又可能创建过多线程，都有内存和资源风险。
>
> 七个参数是核心线程数、最大线程数、空闲存活时间、时间单位、任务队列、线程工厂和拒绝策略。执行顺序是先创建到核心线程数，再入队，队列满后扩到最大线程数，最后拒绝。Android 中还要注意，CallerRunsPolicy 可能把任务交回主线程执行。

![线程池任务分配流程](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/interview12_1.png)

## 3. Android多进程有没有用过？为什么要把组件放到独立进程？

### 原理梳理

在 Manifest 给组件加 `android:process=":remote"`，就能让它运行在应用的独立私有进程中。

#### 为什么把组件放到独立进程

常见理由是隔离不稳定的原生模块、第三方 SDK 或需要独立提供能力的服务。某个进程崩溃后，其他进程通常仍能运行，但依赖这个服务的功能需要处理失败并恢复连接。

#### 多进程带来的问题

- **初始化重复。** 每个进程会创建自己的 Application，需要按进程选择初始化哪些 SDK。
- **数据不共享。** 每个进程有独立的堆、单例和静态变量。交换数据需要 IPC 或支持多进程的存储方案，普通 SharedPreferences 不适合跨进程同步。
- **开销增加。** 多了进程启动、内存占用和 IPC 的成本，还要处理服务断开、重新绑定和回调注销。

普通耗时任务用工作线程即可。Service 即使运行在独立进程中，也不能堵住该进程的主线程。

### 面试口述

先记住：**独立进程用于隔离；内存也隔离，所以单例不共享，数据要通过 IPC 传。**

> 多进程是在 Manifest 给组件配置 android:process，比如冒号 remote。主要是为了隔离不稳定的 SDK、原生模块或独立服务，让它们崩溃时尽量不影响主页面。
>
> 但每个进程都有自己的内存，Application 会分别创建，单例和静态变量也不共享。开发时要按进程初始化，通过 IPC 交换数据，还要处理连接断开和额外内存开销。普通耗时任务用后台线程就够了，不需要专门开进程。

## 4. 图片库用过哪些？加载图片怎么做？

### 原理梳理

#### 常见图片库

常见库有 Glide、Coil、Picasso、Fresco。Glide 在 View 项目里使用广，支持缓存、变换、预加载和生命周期管理；Coil 基于 Kotlin，提供协程和 Compose 集成。面试时先说实际用过的库，再选其中一个解释流程。

#### 以 Glide 为例，看一次请求怎么完成

```kotlin
Glide.with(fragment)
    .load(url)
    .placeholder(R.drawable.placeholder)
    .error(R.drawable.error)
    .into(imageView)
```

1. `with(fragment)` 绑定 Fragment 生命周期，`load` 指定来源，`into` 指定目标。传 Application Context 则没有页面级生命周期绑定。
2. 查活动资源和内存缓存。活动资源是仍在使用的图片资源；内存缓存保存可再次利用的结果。命中后可以省去读取、解码和变换。
3. 查磁盘缓存。Glide 区分变换后资源缓存和原始数据缓存，实际查找、写入范围由 `DiskCacheStrategy` 决定。
4. 未命中则访问网络、本地文件、content URI 等数据源。网络是数据来源，不属于缓存。
5. 在后台按目标尺寸降采样解码，再做裁剪、圆角等变换。
6. 解码路径允许时从 BitmapPool 取得可复用内存，减少分配和 GC。
7. 回到主线程更新目标，由 Glide 管理资源的使用、缓存和回收。

#### 采样、缓存和内存复用有什么区别

采样减少像素数量，缓存保存图片内容，BitmapPool 复用内存空间。同一个头像反复展示可以命中缓存；不同头像也能使用内存池，减少分配。

缓存 key 不只包含 URL，尺寸、变换、签名等也会影响命中。`inSampleSize`、`inBitmap` 是理解采样和复用的入口，但具体解码器、系统版本和硬件 Bitmap 会改变实现路径，不能说所有图片都执行相同的一套 API。

4000×3000 的 ARGB_8888 位图像素数据约占 46 MiB。即使下载文件很小，完整解码也可能 OOM，因此要在解码时按需采样。列表还要处理旧请求和占位图；是否暂停滑动中的请求，应根据卡顿和空白图表现评估。

细节依据 Glide 的[缓存说明](https://bumptech.github.io/glide/doc/caching.html)、[资源复用说明](https://bumptech.github.io/glide/doc/resourcereuse.html)和[解码实现](https://github.com/bumptech/glide/blob/master/library/src/main/java/com/bumptech/glide/load/resource/bitmap/Downsampler.java)。

### 面试口述

先记住：**先找缓存，没有再读数据；按显示尺寸解码，回主线程显示。缓存省重复加载，内存池省重复分配。**

> 常见图片库有 Glide、Coil、Picasso 和 Fresco。Glide 支持缓存、图片变换和生命周期管理，Coil 对 Kotlin 和 Compose 集成比较方便。以 Glide 为例，图片先查活动资源和内存缓存，再查磁盘缓存，都没命中才从网络或本地读取。然后按目标 View 的大小解码，做裁剪或圆角，最后回主线程显示。
>
> 优化时主要注意按需采样，避免大图 OOM；利用 BitmapPool 减少内存分配和 GC；请求绑定页面生命周期，页面退出后取消，列表复用时处理旧请求和占位图。

![Glide 图片加载流程](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/interview12_4.png)

## 5. WebView如何拦截跳转？怎么节约网络资源，避免重复加载资源？

### 原理梳理

#### 跳转和资源请求分别处理

页面跳转用 `WebViewClient.shouldOverrideUrlLoading()`：站内 `http/https` 返回 `false`，让 WebView 自己继续加载；需要 App 接管的链接，处理完成后返回 `true`。例如电话、支付等自定义 scheme，先检查白名单和目标是否可用，再尝试启动 Intent。

不要在回调里对同一个 URL 再次调用 `loadUrl()`。这个回调也不覆盖 POST 和 App 主动调用的 `loadUrl()`。

#### 怎样减少重复加载

1. **先减少不必要的请求。** 页面重建、重复订阅或初始化可能触发多次 `loadUrl`，先排查请求入口。前端也要避免重复引入脚本、图片和样式。
2. **让可缓存资源复用。** 新鲜缓存直接使用；过期资源可通过 ETag/Last-Modified 协商，未变则返回 304，减少响应体传输。304 仍有网络请求，不能等同于完全离线命中。
3. **提供本地资源。** 通过 `shouldInterceptRequest()` 返回 `WebResourceResponse`，让部分 JS、CSS、图片使用本地文件。回调在非 UI 线程执行，需处理 MIME、编码、响应头和流关闭；它不覆盖所有形式的请求。

不要为了提速强制使用过期缓存。登录态、个性化页面和公共静态资源要采用不同策略，否则容易出现数据过期或串用户的问题。

### 面试口述

先记住：**跳转用 shouldOverrideUrlLoading，资源替换用 shouldInterceptRequest；少发请求，多用缓存，更新时让旧资源失效。**

> 拦截跳转用 WebViewClient 的 shouldOverrideUrlLoading。返回 false 就交给 WebView 继续加载，返回 true 表示 App 自己处理，比如打开电话或支付页面。这里不要再对同一个地址调用 loadUrl，避免重复发起加载。
>
> 节约网络，一是避免页面重复初始化，二是利用 HTTP 缓存，三是在有离线资源时，通过 shouldInterceptRequest 返回本地 JS、CSS 或图片。缓存要有更新机制，尤其登录态和个性化内容不能一直用旧缓存。

## 6. 缓存如何控制大小，防止内存/存储无限扩张？自己实现LRU缓存怎么用Java集合实现？

### 原理梳理

#### 先限制大小，再考虑过期

内存缓存设最大字节数，超限淘汰，并响应系统内存压力。图片大小差别很大，不能只限制“100 张”。磁盘缓存限制总大小和文件数量，同时清理过期内容与临时文件。TTL 只规定失效时间，仍需执行清理才能释放空间。

#### 用 Java 集合实现 LRU

LRU 淘汰最久没有访问的数据。可以用 HashMap 加双向链表实现，查找和移动通常都是 O(1)。Java 的 `LinkedHashMap` 已经包含这两部分：将 `accessOrder` 设为 `true`，访问过的元素移到队尾，超限时删除队首。

```java
Map<String, Bitmap> cache = new LinkedHashMap<String, Bitmap>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, Bitmap> eldest) {
        return size() > 100;
    }
};
```

示例按条目数淘汰，且没有同步保护。Android 图片缓存通常用 `LruCache`，重写 `sizeOf` 按 `Bitmap.byteCount` 计费。

### 面试口述

先记住：**容量限制解决“存太多”，过期清理解决“存太久”；LRU 淘汰最久没访问的那项。**

> 控制缓存要同时设置容量上限和过期策略。内存超限就淘汰，磁盘还要清理过期文件和临时文件。图片按实际字节数算，不能只限制张数。
>
> Java 实现 LRU 可以用 LinkedHashMap，把 accessOrder 设成 true，最近访问的放到队尾，超出容量就通过 removeEldestEntry 删除队首最旧的数据。多线程访问要加同步保护。Android 项目里也可以直接用 LruCache，重写 sizeOf 按字节计费。

![LRU 缓存访问与淘汰示意图](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/interview12_2.png)

## 7. Android跨进程通信有哪些方式？AIDL底层核心是什么？

### 原理梳理

常见 IPC 有 Binder/AIDL、Messenger、ContentProvider、广播和 Socket。AIDL 适合跨进程、需要多接口或并发调用的服务；同一进程内绑定服务直接返回自定义 Binder 更简单。Messenger 更适合串行消息；ContentProvider 适合对外提供结构化数据；Socket 适合跨设备或跨平台。

#### AIDL 调用怎么到达另一个进程

客户端 Proxy 将参数写进 `Parcel`，通过 Binder 事务发送。服务端 Stub 根据方法编号读取参数、调用实现，再写回结果。AIDL 支持基本类型、String 等，自定义数据对象通常实现 `Parcelable`。

#### 实际接入和注意事项

定义 `.aidl` 接口，服务端实现生成的 Stub，并在 `onBind()` 返回；客户端 `bindService()`，在 `onServiceConnected()` 中通过 `Stub.asInterface()` 取得接口。跨进程场景通常得到代理，同进程则可能直接取得本地实现。

远程调用通常在服务端 Binder 线程池中执行，共享数据要保证线程安全。普通调用默认同步，耗时接口不应在主线程调用；`oneway` 可表达异步调用。还需校验权限、处理服务死亡、注销回调；大文件可通过文件描述符传递，避免超出事务大小限制。

### 面试口述

先记住：**AIDL 定接口，Binder 做通信；Proxy 打包，Stub 解包，再调用服务端实现。**

> 常见方式有 AIDL、Messenger、ContentProvider、广播和 Socket。简单串行消息可以用 Messenger，对外共享数据用 ContentProvider，需要跨进程接口调用时可以用 AIDL。
>
> AIDL 底层核心是 Binder。客户端 Proxy 把参数写进 Parcel，通过 Binder 传到服务端；服务端 Stub 读出参数，调用真正的方法，再把结果返回。普通远程调用默认同步，耗时调用不要放在主线程；服务端可能同时处理多个请求，要保证线程安全，传输的数据量也不能太大。

![AIDL 与 Binder 跨进程调用流程](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/interview12_3.png)

## 8. 数据库有没有用过？原子更新怎么做？Cursor不关闭会内存泄漏吗？

### 原理梳理

#### 数据库用什么，原子更新解决什么问题

Android 中常用 SQLite，Room 是它上面的访问封装，提供 SQL 检查、对象映射和响应式查询。

原子更新的意思是相关操作要么完成，要么失败，不能只完成一半。对于计数，还要避免并发丢失更新：两次请求都查到 10，各自在应用里加 1 后写回 11，最后就少加了一次。把计算交给数据库，例如 `UPDATE counter SET value = value + 1 WHERE id = ?`，可以避免这类分开的读改写。

以扣库存为例，把业务条件一起写进 SQL：

```sql
UPDATE product
SET stock = stock - 1
WHERE id = ? AND stock > 0;
```

随后检查受影响行数：1 表示扣减成功，0 可能是库存不足或记录不存在。若还要写订单，两步放入同一个事务，任意一步失败就回滚。事务内避免网络请求等耗时工作，减少占锁时间。

Room 可以用 `@Transaction` 或事务 API 包住相关操作；直接操作 SQLite 时，要在 `finally` 中结束事务，只在全部成功后标记提交。

#### Cursor 不关闭会怎样

Cursor 可能持有 CursorWindow 等底层内存和资源，不关闭会有泄漏风险。Kotlin 用 `use {}`，Java 用 try-with-resources。Room 返回普通实体时，内部 Cursor 通常由生成代码管理；自己取得的 Cursor 仍要关闭。

### 面试口述

先记住：**一次更新尽量交给 SQL，多步操作用事务，用完 Cursor 要关闭。**

> Android 数据库常用 SQLite 和 Room，Room 是 SQLite 上面的封装。原子更新尽量用一条 SQL 完成，比如计数直接写 count 等于 count 加一，避免先查出来再写回造成并发丢失。如果是扣库存加写订单这类多步操作，就放进事务，全部成功才提交，失败就回滚。
>
> Cursor 不关闭会有内存和资源泄漏风险，所以查询结束要关闭，Kotlin 可以用 use 自动释放。Room 返回普通实体时内部 Cursor 由框架管理，自己拿到的 Cursor 仍要自己关闭。

## 9. ContentProvider了解吗？查询媒体数据完整流程是什么？

### 原理梳理

#### ContentProvider 是做什么的

它是 Android 四大组件之一，通过 URI 提供统一的数据访问接口。调用方使用 ContentResolver 做查询、插入、更新、删除，Provider 再访问自己的数据库或文件；跨进程通信细节由框架处理。它也能按权限和临时 URI 授权控制访问范围。

#### 从查询到显示图片的完整流程

1. 确认需求。只让用户选几张图片，用 Photo Picker，直接取得授权后的 URI；要做完整相册列表，再考虑 MediaStore 查询。这是两种入口。
2. 检查访问范围。Android 10+ 访问自己创建的媒体通常无需存储权限；读取其他应用的媒体，要结合系统和 targetSdk 处理权限。Android 13 引入图片、视频、音频分类权限；Android 14+ 还要考虑用户只授权部分照片的情况。
3. 在后台线程调用 `ContentResolver.query()`，指定集合 URI、projection 列、筛选条件和排序。只取需要的列，筛选参数使用 selectionArgs。
4. 遍历 Cursor，读取 `_ID`、名称、时间等；通过 `ContentUris.withAppendedId()` 将 ID 拼到集合 URI，得到单项 content URI。用 `use` 关闭 Cursor。
5. 将 URI 交给图片库或用 `openInputStream()` 读取，读取后关闭流；做好媒体被删除、权限被撤销时的失败处理。

底层跨进程请求通常经 Binder 到 MediaProvider，查询媒体索引并返回结果。这里拿到的是元数据，图片内容还需通过 URI 加载。

### 面试口述

先记住：**ContentResolver 发起查询，MediaStore 提供媒体数据；读 Cursor 拿 ID，拼 content URI 加载。**

> ContentProvider 是 Android 四大组件之一，主要用来向其他应用提供受权限控制的数据访问。调用方通过 ContentResolver 和 URI 查询，系统媒体库就是典型例子。
>
> 查询媒体先处理访问权限，再选 MediaStore 对应的图片、视频或音频 URI，调用 ContentResolver.query，指定查询列、条件和排序。拿到 Cursor 后读取 ID、名称等信息，用 ID 拼出 content URI，交给图片库或 ContentResolver 读取，最后关闭 Cursor。查询在后台执行；只需要选几张图片时，可以直接用系统 Photo Picker。

## 10. HTTP和HTTPS的区别？常用HTTP方法有哪些？GET和POST区别？

### 原理梳理

#### HTTP 和 HTTPS 的区别

普通 HTTP 不提供传输加密；HTTPS 通过 TLS 提供加密、完整性保护和服务端身份认证。客户端校验证书链和主机名，通过握手协商密钥，再用对称加密保护业务数据。

#### 常用方法有哪些

- GET：读取资源。
- POST：提交数据，交给资源处理，也常用于创建。
- PUT：创建或整体替换目标资源的状态。
- PATCH：局部更新。
- DELETE：删除目标资源。
- HEAD：与 GET 类似，但不返回响应体。
- OPTIONS：查询通信选项，浏览器预检请求也会用到它。

#### GET 和 POST 怎么区分

GET 用于获取资源，语义上安全且幂等；POST 用于提交处理，协议不保证安全或幂等。这里“安全”指不请求改变资源状态，“幂等”指重复执行的预期效果与一次相同，和有没有加密是两回事。

GET 参数常放查询串，POST 数据常放请求体，但 POST 也可带查询参数。参数位置和长度限制都不是根本区别，实际长度限制取决于客户端、服务器和代理。敏感数据仍需 HTTPS，订单等写操作还要防止重复提交。

### 面试口述

先记住：**HTTPS 用 TLS 保护传输；GET 读，POST 交，PUT 整体改，PATCH 局部改，DELETE 删。**

> HTTP 不提供传输加密，HTTPS 在通信中加入 TLS，提供加密、防篡改和服务端身份校验。常用方法有 GET 获取、POST 提交、PUT 整体替换、PATCH 局部更新、DELETE 删除，还有只取响应头的 HEAD 和查询通信选项的 OPTIONS。
>
> GET 和 POST 的主要区别是语义：GET 用于读取，应该安全、幂等；POST 用于提交处理，重复请求可能创建多份订单。GET 参数通常在 URL，POST 数据通常在请求体，但 POST 并不自带加密，敏感数据仍然要用 HTTPS。

## 11. OkHttp、Retrofit有没有用过？怎么获取HTTP通信报文？统一添加公共Header/Cookie怎么做？

### 原理梳理

#### 两个库分别负责什么

OkHttp 是 HTTP 客户端，负责连接、请求、缓存和拦截器；Retrofit 根据接口注解组装请求，再通过 OkHttp 发出，并配合转换器把响应转成业务对象。App 通常复用 OkHttpClient 和 Retrofit，避免反复创建连接池和重复配置。

#### 怎么获取 HTTP 通信报文

开发时使用 `HttpLoggingInterceptor`，按 BASIC、HEADERS、BODY 等级记录请求和响应；也可使用 Android Studio Network Inspector 或代理抓包工具。日志不等于原始 TCP/TLS 抓包，不能打印 token、Cookie、密码等敏感信息。

应用拦截器看到一次逻辑调用；网络拦截器更接近每次真实网络交互，重定向、重试可能产生多次调用，纯缓存命中则不经过网络拦截器。

直接调用 `ResponseBody.string()` 会消耗响应体，后续转换器可能读不到。应使用受限预览或日志组件，避免整段读取大文件。HTTPS 代理抓包还受证书信任和证书固定策略影响。

#### 公共 Header 和 Cookie 放在哪里

公共 Header 用应用拦截器统一添加，例如语言、客户端版本。`header()` 会覆盖同名 Header，`addHeader()` 会追加；认证信息还要确认请求发往可信域名。

Cookie 使用 `CookieJar`：响应 `Set-Cookie` 后通过 `saveFromResponse()` 保存，下次请求通过 `loadForRequest()` 返回匹配 Cookie。存储、过期、域名和路径匹配由具体实现负责，OkHttp 默认不会替我们保存 Cookie。是否持久化也要自行决定。

### 面试口述

先记住：**Retrofit 定义接口，OkHttp 发请求；日志看报文，拦截器加 Header，CookieJar 管 Cookie。**

> Retrofit 用注解定义接口、组装请求和转换响应，底层通过 OkHttp 完成网络通信，两者通常一起使用。
>
> 查看报文可以用 HttpLoggingInterceptor，或者用 Network Inspector、代理抓包，敏感信息要脱敏。公共 Header 放在 OkHttp 拦截器里统一加，header 覆盖同名值，addHeader 是追加。Cookie 通过 CookieJar 的实现接收、保存，再按域名、路径和有效期匹配后带到下次请求中，需要重启后保留就做持久化。

## 12. HTTP状态码404、500、503分别代表什么？

### 原理梳理

#### 状态码含义

**404 Not Found：找不到请求的资源。** 例如 URL 写错、图片被删除、接口路由不存在。它属于 4xx 客户端错误类，但不能仅据此认定客户端代码有错。

**500 Internal Server Error：服务器内部发生错误，无法完成请求。** 例如接口处理中出现未处理异常。它属于 5xx，代码异常是常见原因，并非唯一原因。

**503 Service Unavailable：服务暂时不可用。** 常见于过载、维护或重启期间，等待后可能恢复，也属于 5xx。响应可能来自网关，不代表后端服务器本身正常。

#### 用同一个场景区分

打开商品详情时：

- 商品不存在，可能返回 **404**：找不到。
- 查询商品时服务端发生意外异常，可能返回 **500**：处理出错。
- 服务正在维护或负载过高，暂时接不了请求，可能返回 **503**：暂时不可用。

状态码含义依据 [HTTP 标准 RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html#name-status-codes)。

#### 追问：能不能重试

503 可能带 `Retry-After`，提示多久后再试。读取类请求可以考虑有限次重试；订单、支付等写操作要先确认幂等性和执行结果。404 通常先检查地址和资源，重复请求同一地址未必有用。

### 面试口述

先记住：**404 找不到资源，500 内部出错，503 暂时不可用。404 属于 4xx，500、503 属于 5xx。**

> 404 表示请求的资源不存在，比如地址写错、资源被删除或路由不存在，属于 4xx 客户端错误类。500 是服务器内部错误，比如处理请求时发生了未处理异常。503 是服务暂时不可用，常见于过载、维护或重启，等一会儿可能恢复。500 和 503 都属于 5xx 服务端错误类。

## 写在最后

涉及“用过什么”的题目，补上自己的真实项目；没有用过，就说明了解到哪一步。

复习时先抓住每题主线，再准备一个代码或实践例子，方便接住追问。

加油！⛽️

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**
