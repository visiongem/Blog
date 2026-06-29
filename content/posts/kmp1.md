---
title: "KMP 学习笔记（一）：项目创建与结构概览"
date: 2026-06-29T21:00:00+08:00
draft: false
categories: ["KMP"]
tags: ["Kotlin", "KMP", "Compose Multiplatform"]
---

今天开始 Kotlin Multiplatform 的学习。第一件事，把项目跑起来，然后搞清楚里面有什么。

## 一、准备工作：AS 和插件

首先，检查工具 Android Studio 是否升级到最新的版本，目前我笔记本的 Android Studio 版本，如下图：

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/kmp0_1.png)

下载个 KMP 插件：

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/kmp0_2.png)

## 二、创建项目

然后 File → New → New Project，选择 Kotlin Multiplatform：

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/kmp0_3.png)

Next，随便取名啥的，我叫它 ChatKmp：

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/kmp0_4.png)

平台嘛，为了都了解下学习了，干脆全部勾选了——Android、iOS、Desktop、Web（JS + Wasm）、Server 全上：

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/kmp0_5.png)

项目创建成功后，可以看到项目结构如图：

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/kmp0_6.png)

乍一看模块不少，别急，一个一个来。

## 三、项目结构：每个模块是干嘛的

AS 生成的 KMP 模板项目，顶层长这样：

```
ChatKmp/
├── build.gradle.kts          # 根 build 脚本，声明插件（不 apply）
├── settings.gradle.kts       # 模块注册 + 仓库配置
├── gradle.properties         # Kotlin / Gradle / Android 全局属性
├── gradle/libs.versions.toml # 版本目录（统一管理所有依赖版本）
├── core/                     # 纯共享逻辑层（无 UI）
├── app/
│   ├── shared/               # Compose Multiplatform 共享 UI 层
│   ├── androidApp/           # Android 壳工程
│   ├── desktopApp/           # Desktop 壳工程（JVM + Compose Desktop）
│   ├── webApp/               # Web 壳工程（JS / Wasm）
│   └── iosApp/               # iOS 壳工程（Swift + Xcode）
└── server/                   # Ktor 服务端（JVM）
```

整个项目的依赖关系，两路汇聚到 core：

```
androidApp / desktopApp / webApp / iosApp
    └── app:shared ──→ core ←── server
```

各平台壳工程**只依赖** `app:shared`，`app:shared` 依赖 `core`，`server` 也依赖 `core`。`core` 是最底层，不依赖任何项目内模块。

### 3.1 core —— 纯逻辑，不沾 UI

`core` 模块的 build.gradle.kts 声明了五个目标平台：

```kotlin
kotlin {
    iosArm64(); iosSimulatorArm64()  // iOS
    jvm()                             // Desktop / Server
    js { browser() }                  // JS
    wasmJs { browser() }             // Wasm
    androidLibrary { ... }            // Android
}
```

它里面目前只有一个文件——`GreetingUtil.kt`：

```kotlin
package com.nia.chatkmp

fun sayHello(to: String): String =
    "Hello, $to!"
```

一个纯 Kotlin 函数，没有任何平台依赖。`core` 的定位就是放这种**所有平台都能用的业务逻辑**——数据模型、工具函数、网络层接口、数据库抽象等等，只要不碰 UI，就往 `core` 里放。

### 3.2 app:shared —— Compose Multiplatform 共享 UI

这是整个项目最核心的模块。它的 build.gradle.kts 不仅声明了多平台目标，还引入了 Compose Multiplatform 插件：

```kotlin
plugins {
    alias(libs.plugins.kotlinMultiplatform)
    alias(libs.plugins.androidMultiplatformLibrary)
    alias(libs.plugins.composeMultiplatform)   // ← 跨平台 UI
    alias(libs.plugins.composeCompiler)         // ← Compose 编译器
}
```

`shared` 模块里有几个关键文件，正好把 KMP 最核心的机制展示了一遍——这个放到第四节单独说。

### 3.3 壳工程 —— 每个平台一个入口

| 模块 | 入口文件 | 关键依赖 |
|------|---------|---------|
| `androidApp` | `MainActivity.kt` | `App()` 来自 shared |
| `desktopApp` | `main.kt` | `App()` 来自 shared |
| `webApp` | `main.kt` | `App()` 来自 shared |
| `iosApp` | `ContentView.swift` | 桥接 `MainViewController` 来自 shared |

每个平台的入口做的事几乎一样——调用 shared 里定义的 `App()` Composable 函数，然后各自平台的启动方式不同：

**Android** —— 标准的 `ComponentActivity` + `setContent { App() }`：

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        enableEdgeToEdge()
        super.onCreate(savedInstanceState)
        setContent { App() }
    }
}
```

**Desktop** —— Compose Desktop 的 `Window`：

```kotlin
fun main() = application {
    Window(onCloseRequest = ::exitApplication, title = "ChatKmp") {
        App()
    }
}
```

**Web** —— `ComposeViewport`（Compose for Web 的入口容器）：

```kotlin
fun main() {
    ComposeViewport { App() }
}
```

**iOS** —— SwiftUI 桥接 Compose UIViewController：

```swift
struct ContentView: View {
    var body: some View {
        ComposeView().ignoresSafeArea()
    }
}
```

`ComposeView` 包装了 shared 模块里 `MainViewController()` 返回的 `UIViewController`——那是 Compose Multiplatform 在 iOS 上暴露 UI 的方式。

### 3.4 server —— 独立的 Ktor 后端

除了四个 UI 壳工程，模板还生成了一个 `server` 模块，不依赖 shared，只依赖 core：

```kotlin
fun main() {
    embeddedServer(Netty, port = 8080, host = "0.0.0.0", module = Application::module)
        .start(wait = true)
}

fun Application.module() {
    routing {
        get("/") {
            call.respondText(sayHello("Ktor"))
        }
    }
}
```

直接复用了 `core` 里的 `sayHello()`。server 跟 UI 毫无关系，但它同样享受 core 的共享逻辑——这就是 KMP 想做的事情：**一套逻辑，到处使用**，不管前端还是后端。

## 四、expect / actual：KMP 怎么做到"写一次，到处跑"

### 4.1 先看一个具体问题

模板项目里有个需求：App 启动后想显示一条问候语 "Hello, Android 36" 或 "Hello, iOS 18.0"——名字前面那部分是固定的，但后面的平台信息每个系统都不一样。

最简单的想法：在 commonMain 里写个 `if` 判断平台？

```kotlin
// ❌ 这样写不行——编译直接失败
fun getPlatformName(): String {
    return when {
        isAndroid() -> "Android ${Build.VERSION.SDK_INT}"  // isAndroid() 不存在，Build 也不存在
        isIOS()     -> "iOS ${UIDevice.currentDevice.systemVersion}" // isIOS() 不存在，UIDevice 也不存在
        else        -> "Unknown"                            // 连 else 也是多余的，上面全都不合法
    }
}
```

不仅 `Build`（Android SDK）和 `UIDevice`（iOS SDK）在 commonMain 里不可见——就连 `isAndroid()` / `isIOS()` 这种判断平台的方法，commonMain 里也没有。Kotlin 标准库不提供"当前是什么平台"的运行时 API。所以这条"在 commonMain 里 if-else 判断平台"的路从头就走不通。

### 4.2 expect：我定规矩，你们各自想办法

KMP 的做法：在 commonMain 里只写**函数签名**，不写函数体。加一个 `expect` 关键字：

```kotlin
// shared/commonMain — Platform.kt
interface Platform {
    val name: String
}

expect fun getPlatform(): Platform   // ← 只有签名，没有 body
```

`expect` 翻译过来就是："**有一个函数叫 `getPlatform()`，返回一个 `Platform`。但是它怎么实现，我不知道，也不该我知道——每个平台自己去写。**"

注意：`expect` 声明**只有签名，没有 `{}` 函数体**。如果你试图给 expect 写 body，编译器直接报错。

### 4.3 actual：每个平台交自己的作业

然后在每个平台的源码目录下，用 `actual` 补上实现：

```
shared/
├── commonMain/   → expect fun getPlatform(): Platform    （定规矩）
├── androidMain/  → actual fun getPlatform(): Platform    （Android 交作业）
├── iosMain/      → actual fun getPlatform(): Platform    （iOS 交作业）
├── jvmMain/      → actual fun getPlatform(): Platform    （Desktop 交作业）
├── jsMain/       → actual fun getPlatform(): Platform    （JS 交作业）
└── wasmJsMain/   → actual fun getPlatform(): Platform    （Wasm 交作业）
```

以 Android 为例，它的 `actual` 长这样：

```kotlin
// shared/androidMain — Platform.android.kt
class AndroidPlatform : Platform {
    override val name: String = "Android ${Build.VERSION.SDK_INT}"
}

actual fun getPlatform(): Platform = AndroidPlatform()
```

iOS 的 `actual`：

```kotlin
// shared/iosMain — Platform.ios.kt
class IOSPlatform : Platform {
    override val name: String = UIDevice.currentDevice.systemName() + " " + UIDevice.currentDevice.systemVersion
}

actual fun getPlatform(): Platform = IOSPlatform()
```

JVM（Desktop）的 `actual`：

```kotlin
// shared/jvmMain — Platform.jvm.kt
class JVMPlatform : Platform {
    override val name: String = "Java ${System.getProperty("java.version")}"
}

actual fun getPlatform(): Platform = JVMPlatform()
```

每个平台各写各的 `actual`，在自己的代码里自由使用各自平台的 SDK（`Build`、`UIDevice`、`System.getProperty`...）。编译器在编译某个平台时，会把对应的 `expect` 和 `actual` 对上。

### 4.4 约束：编译器保证你不会漏

这套机制有一个硬约束：**每一个声明了 target 的平台，都必须为每一个 `expect` 提供一个 `actual`**。如果只写了 Android 的 actual 而没写 iOS 的，编译 iOS 时会直接报错：

```
Expected function 'getPlatform' has no actual declaration in module ...
```

反过来也一样：如果你写了一个 `actual` 但 commonMain 里没有对应的 `expect`，也报错。两边必须一一对应，编译器做检查。

### 4.5 回到业务代码：用起来跟普通函数没区别

在 commonMain 的 `Greeting` 类里，调用 `getPlatform()` 就像调用一个普通函数：

```kotlin
// shared/commonMain — Greeting.kt
class Greeting {
    private val platform: Platform = getPlatform()  // 直接调，不用管当前是哪个平台

    fun greet(): String {
        return sayHello(platform.name)  // "Hello, " + 平台名
    }
}
```

编译 Android 时，编译器把 `getPlatform()` 接到 `androidMain` 的 actual → 返回 `"Hello, Android 36"`。编译 iOS 时，接到 `iosMain` 的 actual → 返回 `"Hello, iOS 18.0"`。编译 Desktop 时，接到 `jvmMain` 的 actual → 返回 `"Hello, Java 17.0.1"`。

**Greeting.kt 这个文件只写了一次，一行都没改，跑在每个平台上结果还各不相同**——这就是 expect / actual 干的事。

### 4.6 一张图收束

```
commonMain（共享代码）
│
│  expect fun getPlatform(): Platform    ← 定义：我需要这个函数
│  class Greeting { getPlatform()... }   ← 使用：直接调用，不问平台
│
└──┬── androidMain:  actual fun getPlatform() = AndroidPlatform()   ← 实现 1
   ├── iosMain:      actual fun getPlatform() = IOSPlatform()       ← 实现 2
   ├── jvmMain:      actual fun getPlatform() = JVMPlatform()       ← 实现 3
   ├── jsMain:       actual fun getPlatform() = JsPlatform()        ← 实现 4
   └── wasmJsMain:   actual fun getPlatform() = WasmPlatform()      ← 实现 5
```

换个生活中的类比：公司总部（commonMain）定了一个岗位——"获取公司名"。总部不关心你是先查数据库、还是调内部 API、还是翻公司章程，给了一个签名就走了。北京分部和东京分部各自用自己的方式完成这个岗位（actual），最后总部拿到的东西格式一样、类型一样，总部不用管底层怎么干的。

**不靠反射，不靠运行时 if-else，所有匹配在编译期完成——expect 写缺的那一笔，每个 actual 都给填上，编译器保证不会漏。**

## 五、Gradle 怎么管的：version catalog 和模块组织

### 5.1 gradle/libs.versions.toml —— 版本统一管理

所有依赖的版本号、group:artifact 坐标都集中在这里，不再散落在各模块的 build.gradle 里。看一眼当前用到的关键版本：

| 组件 | 版本 |
|------|------|
| Kotlin | 2.4.0 |
| Compose Multiplatform | 1.11.1 |
| AGP | 9.0.1 |
| Ktor | 3.5.0 |
| Material3 | 1.11.0-alpha07 |
| Android compileSdk | 36 |

之后加新依赖，先加到这里，再用 `libs.xxx` 引用，全局一个版本来回用。

### 5.2 settings.gradle.kts —— 模块注册

```kotlin
include(":app:androidApp")
include(":app:desktopApp")
include(":app:shared")
include(":app:webApp")
include(":core")
include(":server")
```

一共六个 Gradle 子模块。咦，前面不是一直在讲 iOS 吗？怎么这里没有 `include(":app:iosApp")`？

因为 iOS App 是 Xcode 工程，不走 Gradle。`iosApp/` 目录下的 `.xcodeproj` 和 `.swift` 文件由 Xcode 直接管理，Kotlin 侧 Gradle 只负责把 `:app:shared` 编译成 iOS Framework，Xcode 那边再链接过去。所以 `settings.gradle.kts` 里不需要 include 它——它根本就不是 Gradle 项目。

`enableFeaturePreview("TYPESAFE_PROJECT_ACCESSORS")` 开启后，可以用 `projects.app.shared` 这样的类型安全方式引用模块，告别字符串拼路径。

### 5.3 根 build.gradle.kts —— 插件声明，不 apply

```kotlin
plugins {
    alias(libs.plugins.androidApplication) apply false
    alias(libs.plugins.kotlinMultiplatform) apply false
    // ... 所有插件都加 apply false
}
```

`apply false` 的意思是：**我只是告诉 Gradle 这里有这些插件可用，具体哪个子模块需要，自己 apply**。

为什么这么干？三件事：

1. **根项目不需要这些插件**。根项目不写 Kotlin 也不打 APK，直接 apply 的话 Android 插件会要求根项目配 `android {}` block，没有就报错。
2. **跟 version catalog 配合，版本集中管**。子模块写 `alias(libs.plugins.kotlinMultiplatform)` 就能直接用，不用每处写完整坐标 `id("org.jetbrains.kotlin.multiplatform") version "2.4.0"`，版本号统一走 `libs.versions.toml`。
3. **避免插件在多个子模块的 ClassLoader 里重复加载**，防止类冲突。

## 六、这个模板项目能跑起来吗？

能。最简单的验证方式——跑 Android App：

```bash
cd ChatKmp
./gradlew :app:androidApp:assembleDebug
```

或者直接在 AS 里选 `androidApp` Run Configuration，点运行。出来的界面很简单：一个按钮 "Click me!"，点一下出来 Compose Multiplatform 的 logo 和一段平台信息。

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/kmp0_8.png)

Desktop 也一样：

```bash
./gradlew :app:desktopApp:run
```

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/kmp0_7.png)

Web 稍微麻烦一点（需要启动开发服务器），iOS 需要 Xcode——这些后面学了专门写。

## 七、小结

这第一篇下来，把 KMP 的项目骨架摸了一遍。几个关键点：

1. **`core`** 放纯 Kotlin 逻辑，不碰 UI，所有平台共享。
2. **`app:shared`** 放 Compose Multiplatform 共享 UI，依赖 core，被所有壳工程依赖。
3. **壳工程**（androidApp / desktopApp / webApp / iosApp）只有入口代码，真正的 UI 在 shared 里。
4. **`expect` / `actual`** 是 KMP 的核心机制——共享代码定契约，平台代码给实现，编译期类型安全。
5. **`libs.versions.toml`** 统一管理版本，别在 build.gradle 里散落版本号。
6. **`server`** 模块是独立的 Ktor 后端，直接依赖 core，展示了"一套逻辑到处用"——`sayHello()` 同时被前端和后端调用。

今日份学习先到这。

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**
