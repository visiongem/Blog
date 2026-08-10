# Kotlin 整洁代码指南：写好函数的 8 条黄金法则

## 引言

写出能运行的代码并不难，真正困难的是写出几年之后依然容易维护的代码。

在实际开发中，我们经常会遇到这样的情况：

一个函数刚开始只有十几行，看起来很简单。但随着需求不断增加，验证逻辑、业务判断、数据处理、网络请求、日志记录全部堆进去，最后变成一个几十甚至上百行的“大函数”。

这样的代码短期内可能没有问题，但随着项目变大，维护成本会越来越高。

今天我们就来聊聊 Kotlin 中编写整洁函数的 8 条黄金法则。

这些原则并不只适用于 Kotlin，它们同样适用于 Java、Swift、JavaScript 等其他编程语言。

不过由于 Kotlin 是本文的重点，我们会结合 Kotlin 示例进行说明。

---

# 一、单一职责：一个函数只做一件事

第一条原则来自 SOLID 五大设计原则中的 S：

**Single Responsibility Principle（单一职责原则）。**

简单来说：

> 一个函数应该只有一个职责，也应该只有一个变化的理由。

来看一个反例。

假设有一个用户注册服务：

```kotlin
class RegistrationService {

    fun registerUser(
        email: String,
        password: String
    ) {
        // 验证邮箱

        // 验证密码

        // 密码哈希

        // 创建用户

        // 保存数据库

        // 发送欢迎邮件
    }
}
```

这个函数的问题在哪里？

表面上看，它确实完成了“注册用户”这个任务。

但是仔细观察会发现，它实际上承担了很多不同职责：

* 邮箱格式校验；
* 密码规则校验；
* 密码加密；
* 用户对象创建；
* 数据库存储；
* 邮件发送。

任何一个地方发生变化，都可能导致这个函数需要修改。

比如：

密码规则变化：

> 密码长度从 12 位调整到 16 位。

需要修改这里。

数据库结构变化：

> 用户表增加字段。

也需要修改这里。

邮件系统变化：

> 欢迎邮件改成短信通知。

同样需要修改这里。

这就是典型的“多个变化原因”。

更好的设计应该是让 `registerUser()` 只负责组织流程：

```kotlin
fun registerUser(
    email: String,
    password: String
) {
    validateEmail(email)
    validatePassword(password)

    val hashedPassword = hashPassword(password)

    saveUser(
        email,
        hashedPassword
    )

    sendWelcomeEmail(email)
}
```

现在这个函数读起来非常清楚：

1. 检查邮箱；
2. 检查密码；
3. 加密密码；
4. 保存用户；
5. 发送欢迎邮件。

具体实现细节被拆分到了独立函数中。

这样做的好处：

第一，提高可读性。

别人只需要看函数调用，就能理解整个业务流程。

第二，提高复用性。

例如：

密码校验不仅注册时需要，修改密码、找回密码同样需要。

如果它独立存在：

```kotlin
validatePassword(password)
```

那么其他地方可以直接复用。

第三，更容易测试。

每个小函数都有明确职责，测试范围更加清晰。

所以：

> 一个函数越专注，它越容易理解、修改和复用。

---

# 二、避免副作用：不要让函数偷偷改变外部状态

第二条原则是：

**避免隐藏副作用。**

什么是副作用？

简单来说：

> 一个函数除了返回结果之外，还修改了函数之外的状态。

例如：

```kotlin
fun applyDiscount(order: Order): Price {

    val price = calculateDiscount(order)

    couponRepository.markAsUsed(order.coupon)

    analytics.track(
        "coupon_used"
    )

    return price
}
```

这个函数名字叫：

```kotlin
applyDiscount()
```

按照名字理解，它应该只是计算优惠后的价格。

但是实际上它还做了两件事情：

* 修改数据库中的优惠券状态；
* 发送分析事件。

这就是隐藏副作用。

问题在哪里？

调用者看到：

```kotlin
val price = applyDiscount(order)
```

可能只认为：

> 我计算一下价格。

但实际上：

* 数据库被修改了；
* 数据被发送给分析系统。

如果未来某个地方只是想计算价格，而不希望真正兑换优惠券，就会产生问题。

更好的设计方式：

让计算逻辑保持纯净：

```kotlin
fun calculatePrice(order: Order): Price {

    return applyDiscount(
        order.price,
        order.coupon
    )
}
```

真正提交订单时：

```kotlin
fun placeOrder(order: Order) {

    val price = calculatePrice(order)

    couponRepository.markAsUsed(
        order.coupon
    )

    orderRepository.save(order)
}
```

这样职责更加明确：

* `calculatePrice()` 负责计算；
* `placeOrder()` 负责下单。

当然，副作用并不是完全不能存在。

数据库写入、网络请求、日志记录，这些在真实项目中都是必要的。

关键是：

> 副作用应该出现在名字和职责明确表达它的位置。

例如：

```kotlin
saveUser()
sendEmail()
deleteAccount()
```

这些函数本身就明确表示会产生外部影响。

---

# 三、DRY：不要重复自己

第三条原则：

**Don't Repeat Yourself（不要重复自己）。**

重复代码最大的风险不是代码变长，而是未来修改困难。

例如：

```kotlin
fun formatReceiptPrice(price: Double): String {
    return "%.2f".format(price)
}


fun formatInvoicePrice(price: Double): String {
    return "%.2f".format(price)
}


fun formatRefundPrice(price: Double): String {
    return "%.2f".format(price)
}
```

三个函数里面存在完全一样的逻辑。

现在看起来没问题。

但是如果以后价格格式改变：

例如：

从：

```
12.00
```

改成：

```
$12.00
```

那么三个地方都需要修改。

如果漏掉一个地方，就会出现不一致。

更好的方式：

抽取公共逻辑：

```kotlin
fun formatPrice(price: Double): String {
    return "%.2f".format(price)
}
```

然后：

```kotlin
formatPrice(receipt.price)

formatPrice(invoice.price)

formatPrice(refund.price)
```

这样：

* 代码更少；
* 修改成本更低；
* 测试也只需要覆盖一次。

不过需要注意：

DRY 并不意味着所有重复代码都必须立即抽取。

有时候两个看起来相似的代码，实际上承担不同业务职责。

过早抽象，反而可能让代码变复杂。

正确理解 DRY：

> 不重复业务知识，而不是简单消灭所有重复代码。

# 四、减少函数参数：让函数调用更容易理解

第四条原则：

**尽量减少函数参数数量。**

一个函数需要多少参数，并没有绝对标准。

但是，当一个函数参数越来越多时，通常意味着两个问题：

1. 函数承担了太多事情；
2. 参数之间存在某种关联，却没有被正确表达出来。

来看一个例子：

```kotlin
id="m0jv8q"
fun createInvoice(
    invoiceId: String,
    customerName: String,
    customerEmail: String,
    street: String,
    city: String,
    country: String,
    amount: Double,
    currency: String
) {
    
}
```

这个函数虽然可以工作，但是阅读成本很高。

调用者需要同时理解：

* 哪个参数属于客户信息；
* 哪个参数属于地址信息；
* 哪个参数属于金额信息。

大量独立参数会增加认知负担。

更好的方式，是使用对象表达真实的业务概念。

例如：

```kotlin
id="n5x8kw"
data class Customer(
    val name: String,
    val email: String,
    val address: Address
)


data class Address(
    val street: String,
    val city: String,
    val country: String
)


data class Money(
    val amount: Double,
    val currency: String
)
```

然后：

```kotlin
id="8r7j3m"
fun createInvoice(
    customer: Customer,
    money: Money
) {

}
```

现在函数签名表达的信息更加明确：

创建发票需要：

* 一个客户；
* 一笔金额。

而客户包含什么信息，金额包含什么信息，都由对应的数据类负责。

这样的设计有几个优势。

第一，可读性更高。

调用代码：

```kotlin
id="q7xk2a"
createInvoice(
    customer,
    total
)
```

比一长串字符串和数字参数更加容易理解。

第二，扩展更加容易。

如果未来客户增加会员等级：

```kotlin
id="v3y7pw"
data class Customer(
    val name: String,
    val email: String,
    val address: Address,
    val level: MemberLevel
)
```

调用方不需要改变。

第三，减少参数顺序错误。

例如：

```kotlin
id="2g7n1v"
createUser(
    "Tom",
    "example@test.com"
)
```

如果两个参数都是 String，很容易传反。

而对象类型可以帮助 Kotlin 编译器提前发现问题。

所以：

> 参数越多，不一定代表功能越强，很多时候只是说明缺少更好的抽象。

---

# 五、快速失败：尽早暴露错误

第五条原则：

**Fail Fast（快速失败）。**

它的核心思想：

> 当程序已经确定无法继续执行时，应该立即停止，并明确告诉调用者原因。

很多代码的问题在于：

发现错误之后，仍然继续执行后续逻辑。

例如：

```kotlin
id="8j2f4m"
fun importTransactions(
    file: File
) {

    val data = readFile(file)

    validate(data)

    process(data)

}
```

如果文件本身不存在：

或者格式明显错误：

那么后面的处理没有意义。

更好的方式：

在函数开始阶段进行检查。

Kotlin 中有很多内置工具支持这种写法。

例如：

```kotlin
id="m9q3az"
fun importTransactions(
    file: File
) {

    require(file.exists()) {
        "File does not exist"
    }

    val data = readFile(file)

    process(data)
}
```

`require()` 用于检查调用者传入的数据是否满足要求。

类似的方法还有：

```kotlin
id="s6v8dk"
require(condition)

check(condition)

requireNotNull(value)

checkNotNull(value)
```

它们的作用类似：

* 不满足前置条件，立即失败；
* 错误发生的位置更加靠近根源。

相比让错误一路传播：

最后在很深的位置才崩溃，

快速失败更容易定位问题。

例如：

错误：

```
DatabaseException
```

可能发生在十几个函数调用之后。

而：

```kotlin
require(userId.isNotBlank())
```

直接告诉你：

用户 ID 从一开始就是非法的。

所以：

> 越早发现问题，越容易定位问题。

---

# 六、尽早返回：减少嵌套，让代码更平

第六条原则：

**Early Return（提前返回）。**

在实际项目中，经常会看到这样的代码：

```kotlin
id="t2v8dk"
fun register(
    user: User
): Result {

    if (emailValid(user.email)) {

        if (!exists(user.email)) {

            if (passwordValid(user.password)) {

                createUser(user)

                return Success

            } else {
                return PasswordInvalid
            }

        } else {
            return UserExists
        }

    } else {
        return EmailInvalid
    }
}
```

这段代码的问题不是不能运行。

而是阅读困难。

你需要不断追踪：

* 当前 if 对应哪个 else？
* 哪个条件失败会返回什么？
* 正常流程在哪里？

嵌套越深，理解成本越高。

更好的方式：

使用 Guard Clause（守卫语句）。

```kotlin
id="b9f5hx"
fun register(
    user: User
): Result {

    if (!emailValid(user.email)) {
        return EmailInvalid
    }

    if (exists(user.email)) {
        return UserExists
    }

    if (!passwordValid(user.password)) {
        return PasswordInvalid
    }

    createUser(user)

    return Success
}
```

现在代码结构非常清晰。

阅读顺序：

1. 邮箱错误，结束；
2. 用户存在，结束；
3. 密码错误，结束；
4. 所有条件通过，执行注册。

没有多层缩进，也不需要大量 `else`。

Kotlin 代码中非常推荐这种风格。

例如：

```kotlin
id="h3k8ms"
fun process(user: User?) {

    if (user == null) {
        return
    }

    // 使用 user
}
```

相比：

```kotlin
id="r8q2xa"
fun process(user: User?) {

    if (user != null) {

        // 大量代码

    }
}
```

前者通常更容易阅读。

所以：

> 正常流程保持靠左，异常情况尽早退出。

这也是很多优秀 Kotlin 项目中常见的代码风格。

---

# 七、保持合适的抽象层级：一个函数不要混合不同层次的细节

第七条原则：

**函数内部的代码应该处于相同的抽象层级。**

这是《Clean Code》中非常经典的一个观点：

> 一个函数应该只描述当前层级的业务流程，而隐藏更底层的实现细节。

来看一个反例。

假设我们有一个订单服务：

```kotlin id="x9s7qa"
class CheckoutService {

    fun placeOrder(order: Order) {

        // 验证信用卡号码

        // 计算订单总价

        // 调用支付接口

        // 保存订单

    }
}
```

乍一看，这个函数似乎没有问题。

但是如果继续展开：

```kotlin id="1p8r3m"
fun placeOrder(order: Order) {

    var sum = 0.0

    for (item in order.items) {
        sum += item.price * item.quantity
    }


    var checksum = 0

    for (digit in cardNumber) {
        checksum += digit
    }


    apiClient.post(
        "/orders",
        order
    )
}
```

问题就出现了。

`placeOrder()` 是一个高层业务流程。

它表达的是：

> 用户提交订单。

但是里面却混入了：

* 金额计算算法；
* 信用卡校验算法；
* API 请求细节。

这些都是更低层的实现。

一个好的高层函数应该像这样：

```kotlin id="s8p4r2"
fun placeOrder(order: Order) {

    validatePayment(order)

    calculateTotal(order)

    submitOrder(order)

}
```

现在读代码的人，即使不是开发人员，也能理解：

下单需要：

1. 验证支付；
2. 计算金额；
3. 提交订单。

具体怎么验证银行卡？

怎么计算价格？

怎么调用接口？

这些细节应该隐藏在对应函数内部。

例如：

```kotlin id="8v3zkm"
fun validatePayment(order: Order) {

    checkCardNumber()

    checkBalance()

}
```

再继续深入：

```kotlin id="m9f1cz"
fun checkCardNumber() {

    // Luhn 算法实现

}
```

越往底层，代码越具体。

越往上层，代码越接近业务语言。

这就是合理的抽象层级。

简单来说：

> 高层函数负责表达业务流程，低层函数负责实现具体细节。

---

# 八、合理使用扩展函数：让真正的拥有者承担行为

第八条原则是 Kotlin 特有的：

**在合适的时候使用扩展函数。**

Kotlin 的扩展函数非常方便：

```kotlin id="4a6v5m"
fun String.isEmailValid(): Boolean {

    return contains("@")

}
```

很多开发者看到扩展函数很好用，于是什么都喜欢写成扩展函数。

但扩展函数并不是越多越好。

它真正表达的是：

> 这个操作属于这个对象。

换句话说：

被扩展的对象应该是这个行为真正的拥有者。

来看一个例子。

假设我们需要向 CSV 文件追加一行：

```kotlin id="k3x5hz"
fun appendCsvRow(
    file: File,
    values: List<String>
) {

}
```

这里真正发生变化的是谁？

是：

```kotlin
file
```

因为我们修改的是文件内容。

所以更自然的设计：

```kotlin id="6p2w8f"
fun File.appendCsvRow(
    values: List<String>
) {

}
```

调用：

```kotlin id="d9v3jh"
file.appendCsvRow(values)
```

读起来非常自然：

> 给这个文件追加一行。

File 就是这个操作的主体。

但是如果写成：

```kotlin id="q8w4mn"
fun List<String>.appendTo(
    file: File
) {

}
```

就很奇怪。

因为：

List 并不是这个操作的拥有者。

它只是提供数据。

真正被修改的是 File。

所以判断一个扩展函数是否合理，可以问自己：

> 这个对象是不是这项操作真正的主体？

如果答案是：

是。

那么扩展函数通常比较合适。

如果只是为了少写几个参数：

那可能就是滥用扩展函数。

---

# 总结：写出整洁函数的 8 条原则

最后，我们总结一下今天讲到的 8 条规则。

| 原则           | 核心思想              |
| ------------ | ----------------- |
| 单一职责         | 一个函数只负责一件事情       |
| 避免副作用        | 不要让函数偷偷修改外部状态     |
| DRY          | 不重复业务逻辑           |
| 减少参数         | 用对象表达复杂数据         |
| Fail Fast    | 尽早发现并暴露错误         |
| Early Return | 用提前返回减少嵌套         |
| 抽象层级一致       | 高层函数表达流程，低层函数隐藏细节 |
| 合理使用扩展函数     | 让真正拥有行为的对象承担操作    |

这些原则并不是为了追求所谓“完美代码”。

真实项目中，我们仍然需要根据业务复杂度做取舍。

但是，如果你能在日常开发中坚持这些简单原则：

* 函数会更容易阅读；
* 修改风险会更低；
* 新成员接手代码会更轻松；
* 项目的长期维护成本也会下降。

对于 Android 开发者和 Kotlin Multiplatform 开发者来说，写出整洁代码不仅是一种代码风格，更是一种工程能力。




