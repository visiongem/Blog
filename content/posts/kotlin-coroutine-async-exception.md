---
title: "Kotlin 协程里 async() 的异常，到底什么时候抛？"
date: 2026-06-15T14:00:00+08:00
draft: false
categories: ["Android"]
tags: ["Android", "Kotlin", "协程"]
---

用 `launch` 跑协程，异常一出来父作用域立刻知道；换成 `async`，同样的代码有时静默失败，有时又在 `await()` 处突然炸掉。踩过几次坑之后把行为理了一遍——核心就一句话：`async` 把异常包进 `Deferred`，**什么时候抛**，取决于**有没有**、以及**什么时候**去 `await`。

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/async0.png)

## 一、和 launch 的根本区别

```kotlin
fun main() = runBlocking {
    launch {
        throw IllegalStateException("launch 里的异常")
    }
    delay(100)
}
// 程序崩溃，CoroutineExceptionHandler 或 runBlocking 会报告异常
```

```kotlin
fun main() = runBlocking {
    val deferred = async {
        throw IllegalStateException("async 里的异常")
    }
    delay(100)
    println("还在跑，异常还没冒出来")
    deferred.await()  // 异常在这里才抛出
}
```

| | `launch` | `async` |
|---|---|---|
| 返回值 | `Job` | `Deferred<T>` |
| 异常时机 | 协程体执行时立刻向父作用域传播 | 协程体执行时**不抛**，封装进 `Deferred` |
| 获取异常 | 靠 `CoroutineExceptionHandler` 或父作用域失败 | 靠 `await()` / `awaitAll()`（会抛出）；`getCompletionExceptionOrNull()` + `join()`（不抛出） |

`launch` 是 fire-and-forget 的"启动即负责"；`async` 是"先干活，结果（含异常）稍后取"。

一个常见的混淆点：`async` 的异常**永远不会**进入 `CoroutineExceptionHandler`——因为异常被封装在 `Deferred` 里，handler 只处理 `launch` 这种"无处可去"的未捕获异常。`async` 的异常要主动去 `Deferred` 里取：要么通过 `await()` 让它重新抛出来，要么用 `getCompletionExceptionOrNull()` 不抛异常地检查（见第六节）。

## 二、await() 是主要入口

`async` 块里的异常会被捕获，存进 `Deferred` 的内部状态。调用方**必须**通过 `await()` 才能把异常重新抛出来：

```kotlin
suspend fun fetchUser(): User = coroutineScope {
    async { api.getUser() }.await()
}

// 在 fetchUser 的 await() 外包 try/catch，失败返回 null 而非向上抛
// 等价于：try { fetchUser() } catch (e: Exception) { Log.e(...); null }
suspend fun fetchUserSafe(): User? = coroutineScope {
    val deferred = async { api.getUser() }
    try {
        deferred.await()  // try/catch 必须包在 await() 外，async 块里异常此时才冒出来
    } catch (e: Exception) {
        Log.e("TAG", "请求失败", e)
        null
    }
}
```

两者并不等价：`fetchUser` 失败时抛异常，`fetchUserSafe` 失败时吞掉异常。`try/catch` 只能写在 `await()` 外面——包在 `async { }` 外接不住 `Deferred` 里延迟的异常。

`runCatching` 也常用：

```kotlin
val result = runCatching { async { riskyWork() }.await() }
result.onFailure { Log.e("TAG", "failed", it) }
```

注意：`Deferred` 本身不会"吞掉"异常——它只是**延迟暴露**。不 `await` 的话，异常就一直在 `Deferred` 里等着。

## 三、不 await 会怎样？

这是最容易踩坑的地方。

### 3.1 在 coroutineScope 里

```kotlin
suspend fun loadAll() = coroutineScope {
    async { throw IOException("网络错误") }  // 没有 await
    async { fetchConfig() }.await()
}
// loadAll() 照样失败——coroutineScope 会等所有子协程结束，任一子协程异常都会导致 scope 失败
```

`coroutineScope` 是**结构化并发**：子协程全部完成后父协程才算完成。这里容易有一个误解——没人 `await`，异常怎么跑出来的？

关键点：`async` 启动时同时创建了两样东西——一个 `Deferred<T>` 包着结果等人来取，一个**子 Job** 挂在 `coroutineScope` 的 Job 树上。`IOException` 抛出来时，子协程立即以异常状态结束，子 Job 的失败信号沿 Job 树向上传播。父 Job 收到后：自身标记为失败 + 取消其他正在跑的子协程。`coroutineScope` 收尾时发现兄弟们全停了，把自己保存的异常重新抛给调用方。

整个过程**完全不需要 `await()` 参与**——Job 树的失败传播和 `Deferred` 里等 `await()` 取出异常，是两条并行的线。某个 `async` 即使没人 `await`，它异常完成时 scope 仍然会失败。

### 3.2 在 GlobalScope / 裸 async 里

```kotlin
fun fireAndForget() {
    GlobalScope.async(Dispatchers.IO) {
        throw RuntimeException("没人管我")
    }
    // 函数直接返回，异常可能变成未捕获异常，进程崩溃或静默丢失
}
```

没有结构化父作用域兜底的 `async`，不 `await` 就等于没人领异常。这是典型的反模式——需要后台任务用 `launch` + `SupervisorJob`，或者把 `async` 包在 `coroutineScope` / `viewModelScope` 里。

### 3.3 只 await 其中一个

```kotlin
suspend fun partialAwait() = coroutineScope {
    val a = async { throw Exception("A 挂了") }
    val b = async { delay(500); "B OK" }

    try {
        b.await()
    } catch (e: Exception) {
        // b 没问题，进不来
    }
    // scope 结束时 a 的异常仍会导致 partialAwait 失败
}
```

只 `await` 成功的那个不够——scope 里其他失败的 `async` 照样拖垮整个 scope。

## 四、多个 async：await 顺序与并发

```kotlin
suspend fun loadDashboard() = coroutineScope {
    val userDeferred = async { api.getUser() }
    val feedDeferred = async { api.getFeed() }

    // 串行 await：先 user 后 feed
    val user = userDeferred.await()   // 如果这里抛异常，feed 已被 coroutineScope 取消
    val feed = feedDeferred.await()
    Dashboard(user, feed)
}
```

`userDeferred.await()` 抛异常时，`coroutineScope` 会**取消兄弟协程**（`feedDeferred` 被取消），然后异常向上传播。这是 `coroutineScope` 的默认行为——一个失败，全部取消。

并行 await 的写法：

```kotlin
suspend fun loadDashboardParallel() = coroutineScope {
    val userDeferred = async { api.getUser() }
    val feedDeferred = async { api.getFeed() }

    awaitAll(userDeferred, feedDeferred)  // 任一失败，抛第一个遇到的异常
    Dashboard(userDeferred.getCompleted(), feedDeferred.getCompleted())
}
```

`awaitAll` 会等全部完成；如果多个都失败，抛出的通常是**第一个**被检测到的异常，其他的被 suppress（挂在 `SuppressedException` 里）。调试时如果只看到一条，检查是不是还有被 suppress 的。

`awaitAll` 成功后，用 `getCompleted()` 取结果是安全的——它是非挂起函数，直接从已完成的 `Deferred` 里拿值，比再调 `await()` 更轻量。

## 五、supervisorScope：一个失败不影响其他

业务上常见需求：三个接口并行拉，一个挂了其余照常用。

```kotlin
suspend fun loadDashboardResilient() = supervisorScope {
    val userDeferred = async { api.getUser() }       // 可能失败
    val feedDeferred = async { api.getFeed() }     // 独立
    val adsDeferred = async { api.getAds() }       // 独立

    val user = runCatching { userDeferred.await() }.getOrNull()
    val feed = runCatching { feedDeferred.await() }.getOrDefault(emptyList())
    val ads = runCatching { adsDeferred.await() }.getOrNull()

    Dashboard(user, feed, ads)
}
```

`supervisorScope` 底层是 `SupervisorJob`：子协程异常**不会**取消兄弟协程。但每个 `async` 的异常仍然要各自 `await` + 处理，否则 `supervisorScope` 本身在收尾时仍会报告未处理的失败子协程。

对比：

| 作用域 | 一个 async 失败 | 兄弟协程 |
|---|---|---|
| `coroutineScope` | scope 失败 | 全部取消 |
| `supervisorScope` | 仅该子协程失败 | 继续运行 |

## 六、CancellationException 的特殊待遇

协程被取消时抛的是 `CancellationException`（继承链：`CancellationException → IllegalStateException → RuntimeException → Exception`）。`async` 不会把它当作业务异常包装——`await()` 会原样重新抛出。

关键在于：`catch (e: Exception)` **能**捕获到 `CancellationException`，它是 `Exception` 的子类，从 try/catch 层面没有任何特殊待遇。协程库的"特殊处理"发生在**传播层面**——一个子协程因取消而结束，不会触发 `CoroutineExceptionHandler`，也不会在 `supervisorScope` 里把取消传导给兄弟/父协程。但如果在 `catch (e: Exception)` 里顺手把它吞掉了，协程的取消链路就断了：框架以为子协程还在跑，调用方以为一切正常。

正确做法：**先捕获 `CancellationException` 并 rethrow，再兜住业务异常**。

```kotlin
val deferred = async {
    delay(1000)
    "done"
}
deferred.cancel()
try {
    deferred.await()
} catch (e: CancellationException) {
    throw e  // 必须 rethrow，否则破坏协程取消机制
} catch (e: Exception) {
    // 处理业务异常
}
```

如果代码里写了 `catch (e: Throwable)`（比如为了兼容某些库的异常），要格外小心，别把取消当错误处理了。一个简单的检查：

```kotlin
try {
    deferred.await()
} catch (e: Throwable) {
    if (e is CancellationException) throw e
    // 剩下才是真正的业务异常
}
```

此外，`Deferred` 还提供了不抛异常的检查手段——`getCompletionExceptionOrNull()`。配合 `join()` 使用，既知道它失败了，又不会炸掉当前协程，适合日志/上报场景：

```kotlin
val deferred = async { api.getUser() }
deferred.join()  // 等它完成，不抛异常
val ex = deferred.getCompletionExceptionOrNull()
if (ex != null) {
    Log.e("TAG", "请求失败: ${ex.message}")
}
```

## 七、Android 里的实用模式

### ViewModel 并行请求

```kotlin
class HomeViewModel : ViewModel() {

    fun loadHome() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                val (user, feed) = coroutineScope {
                    val u = async { repository.getUser() }
                    val f = async { repository.getFeed() }
                    u.await() to f.await()
                }
                _uiState.value = UiState.Success(user, feed)
            } catch (e: CancellationException) {
                throw e  // viewModelScope 取消时会产生，必须 rethrow
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message)
            }
        }
    }
}
```

外层 `launch` 兜住 `await` 抛出的异常，内层 `coroutineScope` + `async` 做并行。一个接口失败，另一个自动取消，不会浪费资源。

### 部分失败可降级

```kotlin
viewModelScope.launch {
    supervisorScope {
        val avatar = async { runCatching { repo.getAvatar() }.getOrNull() }
        val profile = async { runCatching { repo.getProfile() }.getOrNull() }
        _uiState.value = ProfileUi(
            avatar = avatar.await(),
            profile = profile.await()
        )
    }
}
```

`supervisorScope` 保证一个接口超时不会把另一个也 cancel 掉；`runCatching` 在每个 `async` 内部消化异常，转成 nullable 结果。

## 八、总结

1. **`async` 延迟异常**——异常存在 `Deferred` 里，靠 `await()` 取出。
2. **`launch` 立即传播**——不等调用方来取。
3. **在 `coroutineScope` 里**，即使不 `await`，失败的 `async` 仍会让 scope 失败。
4. **裸 `GlobalScope.async` 不 `await`** 是异常黑洞，避免使用。
5. **一个失败是否拖垮其他**，取决于 `coroutineScope` 还是 `supervisorScope`。
6. **`CancellationException` 不是业务错误**，捕获后必须 rethrow。

`async` 的设计意图很明确：**并发计算 + 延迟取结果**。异常处理跟着 `await` 走，结构化并发保证不 `await` 也不会在 scope 里悄悄漏掉——前提是待在 scope 里。

---

*基于 kotlinx.coroutines 1.7+ 行为整理。不同版本在 `supervisorScope` 未处理异常的细节上可能有微调，以官方文档为准。*

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**