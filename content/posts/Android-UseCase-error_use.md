---
title: "你的 Android 项目，真的需要 UseCase 吗？"
date: 2026-07-13T10:00:00+08:00
draft: false
categories: [ "Android" ]
tags: [ "Kotlin", "Android" ]
---

> 很多人学完 Clean Architecture，回头就给自己的小项目塞了一堆 `XxxUseCase`。用的时候没觉得多爽，改需求的时候倒是先改三个文件。今天聊聊 UseCase 这个东西——它到底是什么、什么时候该用、什么时候纯属自找麻烦。

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/usecase0.png)

## 一、什么是 UseCase

Clean Architecture 的同心圆图如下图（网上搜的截图）：最外圈 **Frameworks and Drivers**（框架与驱动层），往里是 **Interface Adapters**（接口适配层），再往里是 **Use Cases**（用例层），最中心是 **Entities**（实体层）。图中标注的核心规则是 **Dependency Rule（依赖规则）**——依赖方向始终由外向内，内层不感知外层细节。落到代码上就是一件事：**把「应用行为」从「界面」和「数据来源」里剥离出来。**

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/clean_architecture.png)

**UseCase（用例）就是承载一个用户/系统行为对应业务流程的单元**——它可能包含业务规则、流程编排、状态转换，或者三者的任意组合。

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/usecase1.png)

Entity、UseCase、Repository 三者的分工：

- **Entity**：企业核心规则（「订单总价 = 单价 × 数量 − 优惠」，什么系统都这么算）
- **UseCase**：一个用户目标怎么完成（「下单」要经过查购物车 → 验优惠券 → 算价格 → 创建订单）
- **Repository**：数据从哪来、存哪去（UseCase 不关心）

UseCase 里装的不只是「计算规则」，还有**流程编排、调用顺序控制、数据转换、权限判断、错误处理、状态转换**。它的价值在于「**整合**」而不是「**转发**」——这是区分该用不该用的分水岭。

一个简单但有整合的 UseCase 长这样：

```kotlin
class GetHomeFeedUseCase(
    private val postRepo: PostRepository,
    private val userRepo: UserRepository,
) {
    suspend operator fun invoke(): HomeFeed {
        val posts = postRepo.getLatest()
        val user = userRepo.getCurrentUser()
        return HomeFeed(posts, user)  // 组合两个数据源，而不是转发一个调用
    }
}
```

## 二、"可选"不是「不重要」，而是你的 App 可能根本没复杂到需要它

Google 官方的 [架构指南](https://developer.android.com/topic/architecture) 里，Domain Layer 旁边标了两个字：「**可选**」。

![](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/clean1.png)

很多人扫一眼就翻译成了「可以不用」。但 Google 真正的意思是——**很多 App 根本没有复杂到需要它。**

不是 Domain Layer 没用，是你的项目现在还轮不到它。**「可选」=「有收益再引入」，不是「可有可无」。**

这个误解之所以普遍，还有一个原因：很多人没意识到**客户端和后端的 UseCase，侧重点根本不同。**

- **后端**：UseCase 是企业业务规则的大本营——库存怎么扣、优惠怎么算、事务怎么保证一致性。逻辑重、规则多、计算密集。
- **客户端**：UseCase 更多时候承担的是**流程编排**。不是说客户端没有业务规则（输入校验、离线冲突处理、Diff Merge、价格计算、图片编辑、AI 推理——客户端真正吃逻辑的地方并不少），但更常见的场景是把多个步骤串成一个完整的业务动作。

举个例子——登录。后端只负责验证密码和签发 token，客户端这边却要走完：

```
输入校验 → 调登录接口 → 存 token → 初始化用户信息 → 刷新缓存 → 更新导航状态
```

每一步依赖不同的模块。多个步骤、有先后顺序、有异常分支——这些东西散在 ViewModel 里就是隐患，这时候抽成 UseCase，通常比继续堆在 ViewModel 里更容易维护。

> **客户端不是「没有业务」，而是客户端的 UseCase 以流程编排为主、以计算规则为辅。** 当多个步骤需要被整合成一个可复用、可测试的业务动作时，UseCase 就是合适的载体。

反过来，那些「只有一个步骤、只调一个方法」的场景——才是真正不该用 UseCase 的地方。而这恰恰是很多人学完 Clean Architecture 之后干的事。

## 三、反面教材：为了 UseCase 而 UseCase

来看一段真实项目里常见的代码：

```kotlin
class GetUserNameUseCase(private val repo: UserRepository) {
    suspend operator fun invoke(): String = repo.getUserName()
}

class GetUserAvatarUseCase(private val repo: UserRepository) {
    suspend operator fun invoke(): String = repo.getUserAvatar()
}

class GetUserEmailUseCase(private val repo: UserRepository) {
    suspend operator fun invoke(): String = repo.getUserEmail()
}
```

三个类**没有任何编排、没有任何业务逻辑**，纯粹把 Repository 方法换了个壳。结果就是：ViewModel 构造函数膨胀、DI 配置翻倍、新增一个字段要改三个文件。**样板代码换来的收益是零。**

### 同样是「薄」，为什么有的值得抽？

对比一下「退出登录」：

```kotlin
class LogoutUseCase(
    private val authRepo: AuthRepository,
    private val cacheRepo: CacheRepository,
    private val analyticsRepo: AnalyticsRepository,
) {
    suspend operator fun invoke() {
        authRepo.logout()          // 清除 token
        cacheRepo.clearAll()       // 清空缓存
        analyticsRepo.track("user_logout")  // 上报埋点
    }
}
```

没有复杂计算、没有校验、没有分支——但它值得成为一个 UseCase。因为：

- **「退出登录」是一个完整的业务动作**，不是「调一个 API」
- **编排了三个依赖**（Auth、Cache、Analytics），而不是一行 `repo.xxx()`
- **可能被多处复用**——设置页退出、token 过期强制退出，同一套流程

而 `GetUserNameUseCase` 呢？一行 `repo.getUserName()` 包了一层，不编排任何东西、不含任何规则、也没有复用理由。

**分水岭：UseCase 的价值不在于「逻辑多不多」，而在于「有没有独立存在的理由」——要么有需要编排的流程，要么有值得隔离的业务规则，要么被多处复用。** 三不沾，就是多余的。

## 四、什么时候该用，什么时候不该用

### ✅ 适合用 UseCase

**1. 需要编排多个数据源 / 多个步骤。** 下单是最典型的例子——校验优惠券、算价格、创建订单，跨三个 Repository，涉及规则计算和多步流程。不该散落在每个下单入口里。

```kotlin
class PlaceOrderUseCase(
    private val cartRepo: CartRepository,
    private val couponRepo: CouponRepository,
    private val orderRepo: OrderRepository,
) {
    suspend operator fun invoke(cartId: String, couponCode: String?): Result<Order> {
        val cart = cartRepo.getCart(cartId)
        val discount = couponCode?.let { couponRepo.validate(it, cart.total) } ?: 0
        val finalPrice = (cart.total - discount).coerceAtLeast(0)
        return orderRepo.create(cart.items, finalPrice)
    }
}
```

同样的道理适用于支付流程、首页聚合、内容发布、离线同步——共同点是「一件事要经过好几步、涉及好几个依赖」。

**2. 同一段逻辑被多个 ViewModel 复用。** 比如签到——首页、个人中心、活动页都要用。抽成 `CheckInUseCase`，改一次全部生效。

**3. 逻辑本身够复杂，值得单独测试。** 复杂价格计算、风控规则、状态机——脱离 UI 和网络写纯单元测试，回归时有底。

### ❌ 不该用 UseCase

**1. 纯转发。** 一行 `repo.xxx()` 换个名字，没有任何编排——就是第三节里 `GetUserNameUseCase` 那种。

**2. 只在一个界面用，短期内不会复用。** 等真正出现第二个调用方再重构——那时候你才知道该抽什么。

**3. 中小型 CRUD 项目。** 大部分屏幕就是「拉数据 → 展示 → 提交」，`ViewModel → Repository` 两层足够清晰。

### 🔑 自检方法

犹豫要不要抽的时候，问三个问题：

> 1. 它是在完成一个完整的**业务动作**吗？
> 2. 它需要编排**多个步骤或依赖**吗？
> 3. 它会**被多处复用**吗？

至少满足一个，才值得抽。全不满足——比如 `GetUserNameUseCase`——就是多余的。

## 五、我的观点：按场景来，别按信仰来

上面说的是「客观上什么场景配得上一层 UseCase」。至于我个人——**中小型项目里，UseCase 这一层完全可以不要。** `View → ViewModel → Repository` 三层已经把职责分清楚了。简单的 UI 状态逻辑放 ViewModel，数据访问放 Repository。只有当业务流程逐渐复杂、涉及多个依赖或需要复用时，再考虑抽出 UseCase。

规模大的项目——业务规则复杂、流程多、多人协作——那时候「把应用行为层单独抽离」才真的值得考虑。但即便如此，也应该是「评估过、确认有收益」之后再抽，而不是「一上来就默认要有」。

**架构模式是工具，不是信仰。** 判断一个架构决策好不好，标准从来不是「它符不符合某本书 / 某张图」。

**架构的价值，从来不在于层数，而在于维护成本。好的架构，不是让代码看起来更高级，而是让下一次改需求的时候更轻松。**

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**
