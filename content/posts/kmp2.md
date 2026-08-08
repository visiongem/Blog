---
title: "KMP 学习笔记（二）：Ktor Client 网络请求与序列化"
date: 2026-07-01T21:00:00+08:00
draft: false
categories: ["KMP"]
tags: ["Kotlin", "KMP", "Ktor", "kotlinx.serialization"]
---

上一篇项目新建好了，为了自己学习，每个平台都选了，包括 server。 因此今日份学习如何用 Ktor Client 从 shared 代码里发网络请求，拿到数据，再显示在 Compose UI 上吧。

## 一、加依赖：Ktor Client + kotlinx.serialization

先往 `gradle/libs.versions.toml` 里加版本号：

```toml
[versions]
ktor = "3.5.0"
kotlinx-serialization = "1.8.1"

[libraries]
ktor-client-core = { module = "io.ktor:ktor-client-core", version.ref = "ktor" }
ktor-client-content-negotiation = { module = "io.ktor:ktor-client-content-negotiation", version.ref = "ktor" }
ktor-serialization-json = { module = "io.ktor:ktor-serialization-kotlinx-json", version.ref = "ktor" }
ktor-client-okhttp = { module = "io.ktor:ktor-client-okhttp", version.ref = "ktor" }
ktor-client-darwin = { module = "io.ktor:ktor-client-darwin", version.ref = "ktor" }
ktor-client-cio = { module = "io.ktor:ktor-client-cio", version.ref = "ktor" }
kotlinx-serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "kotlinx-serialization" }

[plugins]
kotlinx-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```

然后在 `app/shared/build.gradle.kts` 里引入：

```kotlin
plugins {
    alias(libs.plugins.kotlinMultiplatform)
    alias(libs.plugins.androidMultiplatformLibrary)
    alias(libs.plugins.composeMultiplatform)
    alias(libs.plugins.composeCompiler)
    alias(libs.plugins.kotlinx.serialization)   // ← 序列化插件
}

kotlin {
    // ... 各 target 不变

    sourceSets {
        commonMain.dependencies {
            implementation(libs.ktor.client.core)
            implementation(libs.ktor.client.content.negotiation)
            implementation(libs.ktor.serialization.json)
            implementation(libs.kotlinx.serialization.json)
        }

        androidMain.dependencies {
            implementation(libs.ktor.client.okhttp)
        }

        iosMain.dependencies {
            implementation(libs.ktor.client.darwin)
        }

        jvmMain.dependencies {
            implementation(libs.ktor.client.cio)
        }

        jsMain.dependencies {
            implementation(libs.ktor.client.cio)   // CIO 也支持 JS target（实验性）
        }
    }
}
```

注意看依赖的写法：
- **commonMain** 里只放跨平台的接口层（`ktor-client-core`、`content-negotiation`）
- **各平台 sourceSet** 里放具体的 HTTP 引擎：Android 用 OkHttp、iOS 用 Darwin、Desktop 用 CIO

这跟第一篇 expect/actual 的思路一模一样——commonMain 定"我要发 HTTP 请求"，具体走什么引擎是各平台自己的事。版本管理也是第一篇讲过的——所有版本号在 `libs.versions.toml` 一次性配好，子模块只写 `libs.xxx` 引用。

## 二、定义数据模型

既然是聊天 App，先建一个最简单的消息模型。放在 `core` 模块，因为纯数据模型不沾 UI、不沾平台——所有模块都能用。

```kotlin
// core/src/commonMain/kotlin/com/nia/chatkmp/model/ChatMessage.kt
package com.nia.chatkmp.model

import kotlinx.serialization.Serializable

@Serializable
data class ChatMessage(
    val id: Long,
    val sender: String,
    val content: String,
    val timestamp: Long
)

@Serializable
data class ChatResponse(
    val success: Boolean,
    val messages: List<ChatMessage>
)
```

`@Serializable` 注解告诉编译器自动生成序列化/反序列化代码。不用手写 `fromJson()` / `toJson()`，不用反射，编译期搞定。跟 expect/actual 一个思路——能编译期确定的就别拖到运行时。

## 三、创建 API 客户端

在 `app/shared` 模块里建一个 API 客户端类：

```kotlin
// app/shared/src/commonMain/kotlin/com/nia/chatkmp/api/ChatApi.kt
package com.nia.chatkmp.api

import com.nia.chatkmp.model.ChatMessage
import com.nia.chatkmp.model.ChatResponse
import io.ktor.client.HttpClient
import io.ktor.client.call.body
import io.ktor.client.plugins.contentnegotiation.ContentNegotiation
import io.ktor.client.request.get
import io.ktor.client.request.post
import io.ktor.client.request.setBody
import io.ktor.http.ContentType
import io.ktor.http.contentType
import io.ktor.serialization.kotlinx.json.json
import kotlinx.serialization.json.Json

class ChatApi {
    private val client = HttpClient {
        install(ContentNegotiation) {
            json(Json {
                ignoreUnknownKeys = true
                isLenient = true
            })
        }
    }

    suspend fun getMessages(): List<ChatMessage> {
        val response: ChatResponse = client
            .get("http://10.0.2.2:8080/messages")   // Android 模拟器专用地址
            .body()
        return response.messages
    }

    suspend fun sendMessage(sender: String, content: String): ChatMessage {
        return client
            .post("http://10.0.2.2:8080/messages") {
                contentType(ContentType.Application.Json)
                setBody(mapOf("sender" to sender, "content" to content))
            }
            .body()
    }
}
```

几个关键点：

**`HttpClient {}` 配置**：用 `install(ContentNegotiation)` 装上 JSON 序列化插件，指定 `ignoreUnknownKeys = true`——服务端多返回了几个字段，客户端不用崩。

**`suspend` 函数**：Ktor Client 的请求都是 suspend 函数，得在协程里调。不用写 callback，挂起等结果，回来了继续往下走。

**Android 模拟器地址 `10.0.2.2`**：这是 Android 模拟器访问宿主机的特殊 IP，映射到宿主机的 `localhost`。真机或 iOS 模拟器需要用真实 IP。

### 3.1 地址怎么处理？——又来一次 expect/actual

上面硬编码了 `10.0.2.2`，每个平台地址不一样。来，再写一套 expect/actual 的 `ServerConfig`：

```kotlin
// shared/commonMain — ServerConfig.kt
expect fun getBaseUrl(): String
```

```kotlin
// shared/androidMain — ServerConfig.android.kt
actual fun getBaseUrl(): String = "http://10.0.2.2:8080"
```

```kotlin
// shared/iosMain — ServerConfig.ios.kt
actual fun getBaseUrl(): String = "http://localhost:8080"
```

```kotlin
// shared/jvmMain — ServerConfig.jvm.kt
actual fun getBaseUrl(): String = "http://localhost:8080"
```

```kotlin
// shared/jsMain — ServerConfig.js.kt
actual fun getBaseUrl(): String = "http://localhost:8080"
```

然后 `ChatApi` 里用 `getBaseUrl()` 替换硬编码地址。第一篇学的 expect/actual，这篇马上用上了——expect/actual 不是学完就扔的概念，每个平台有差异的地方它就会出现。

## 四、在 Compose UI 里展示数据

有了 API 客户端，在 `App()` 里把数据拉出来显示。先改一下第一篇的 `App.kt`：

```kotlin
// shared/commonMain — App.kt
@Composable
fun App() {
    val api = remember { ChatApi() }
    var messages by remember { mutableStateOf<List<ChatMessage>>(emptyList()) }
    var isLoading by remember { mutableStateOf(true) }
    var error by remember { mutableStateOf<String?>(null) }

    LaunchedEffect(Unit) {
        try {
            messages = api.getMessages()
            isLoading = false
        } catch (e: Exception) {
            error = e.message
            isLoading = false
        }
    }

    MaterialTheme {
        when {
            isLoading -> {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    CircularProgressIndicator()
                }
            }
            error != null -> {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    Text("出错了: $error", color = MaterialTheme.colorScheme.error)
                }
            }
            else -> MessageList(messages)
        }
    }
}

@Composable
fun MessageList(messages: List<ChatMessage>) {
    LazyColumn(modifier = Modifier.fillMaxSize().padding(16.dp)) {
        items(messages) { msg ->
            MessageBubble(msg)
        }
    }
}

@Composable
fun MessageBubble(message: ChatMessage) {
    Column(
        modifier = Modifier
            .fillMaxWidth()
            .padding(vertical = 4.dp)
            .background(
                MaterialTheme.colorScheme.surfaceVariant,
                RoundedCornerShape(12.dp)
            )
            .padding(12.dp)
    ) {
        Text(message.sender, style = MaterialTheme.typography.labelSmall)
        Spacer(Modifier.height(4.dp))
        Text(message.content, style = MaterialTheme.typography.bodyMedium)
    }
}
```

跑起来的效果就是一个简单的消息列表：进 App 自动拉数据，加载中转圈，出错显示错误信息，成功就一条一条列出来。

<!-- TODO: 截图 - 消息列表界面 -->

## 五、server 端也要改

server 模块现在还只有一个根路由，返回 `sayHello("Ktor")`。加点 API：

```kotlin
// server/src/main/kotlin/com/nia/chatkmp/Application.kt
fun Application.module() {
    val messages = mutableListOf(
        ChatMessage(1, "系统", "欢迎来到 ChatKmp！", System.currentTimeMillis())
    )

    install(ContentNegotiation) {
        json()
    }

    routing {
        get("/") {
            call.respondText(sayHello("Ktor"))
        }

        get("/messages") {
            call.respond(ChatResponse(success = true, messages = messages))
        }

        post("/messages") {
            val body = call.receive<Map<String, String>>()
            val msg = ChatMessage(
                id = (messages.lastOrNull()?.id ?: 0) + 1,
                sender = body["sender"] ?: "匿名",
                content = body["content"] ?: "",
                timestamp = System.currentTimeMillis()
            )
            messages.add(msg)
            call.respond(msg)
        }
    }
}
```

注意 server 也要把 `core` 模块里的 `ChatMessage` 和 `ChatResponse` 加为依赖——这就是第一篇说的 **"一套数据模型，前端后端一起用"**。`core` 模块的 build.gradle.kts 已经声明了 `jvm()` target，server 直接 `implementation(projects.core)` 就行。

```kotlin
// server/build.gradle.kts
kotlin {
    sourceSets {
        main {
            dependencies {
                implementation(projects.core)   // ← 复用数据模型
            }
        }
    }
}
```

## 六、跑起来验证

分两步：先启 server，再启动客户端。

**启动 server**：

```bash
./gradlew :server:run
```

看到 `Responding at http://0.0.0.0:8080` 就说明 server 跑起来了。先直接用 curl 测试一下：

```bash
# 获取消息列表（目前只有一条欢迎消息）
curl http://localhost:8080/messages

# 发一条新消息
curl -X POST http://localhost:8080/messages \
  -H "Content-Type: application/json" \
  -d '{"sender":"nia","content":"Hello KMP!"}'

# 再取一次，应该看到两条消息
curl http://localhost:8080/messages
```

curl 验证没问题之后，再启动客户端：

- **Android**：`./gradlew :app:androidApp:assembleDebug`，或者 AS 里直接 Run
- **Desktop**：`./gradlew :app:desktopApp:run`

<!-- TODO: 截图 - curl 测试 server API -->

## 七、小结

这篇下来，几个点：

1. **Ktor Client** 加到 shared 模块里，commonMain 写接口，各平台 sourceSet 配引擎（OkHttp / Darwin / CIO）——跟 expect/actual 一样的"定契约 + 各平台实现"模式
2. **kotlinx.serialization** 做 JSON 序列化，`@Serializable` 注解编译期生成代码，不依赖反射
3. **Compose UI 显示数据**：`LaunchedEffect` 起协程拉数据，`mutableStateOf` 驱动 UI 更新，加载 / 错误 / 成功三种状态分支处理
4. **server 端配合**：加 API 路由，复用 `core` 模块的数据模型——真正做到了"一套数据模型前后端共用"

下一篇应该搞 **Koin 依赖注入**，把 `ChatApi` 的实例创建从 Composable 里抽出来，不然后面模块多了到处 `remember { }` 很难维护。

今日份学习先到这。

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**
