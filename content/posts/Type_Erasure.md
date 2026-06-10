---
title: "Java/Kotlin 中什么是类型擦除？为什么它会导致类型不安全？"
date: 2026-06-10T10:00:00+08:00
draft: false
categories: ["Android"]
tags: ["Android", "博客"]
---

> 三个概念，一次讲清楚：类型擦除、类型安全，以及它们之间的"恩怨"。

![封面图](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/classcastexception0.png)

---

## 一、什么是类型擦除（Type Erasure）？

简单说：**泛型只在编译期存在，编译成字节码后，类型参数就被"擦除"了。**

```java
// 你写的代码
List<String> names = new ArrayList<>();
names.add("hello");
String s = names.get(0);

// 编译后等价于
List names = new ArrayList();          // <String> 消失了
names.add("hello");
String s = (String) names.get(0);     // 编译器自动插入了强转
```

Java 选择这样做是为了**向后兼容**——Java 5 引入泛型时，海量的老代码用的还是 `List`（raw type），这些代码必须能继续在新 JVM 上跑。

> Kotlin/JVM 的泛型同样是擦除的，因为它编译成同样的 JVM 字节码。

---

## 二、什么是类型安全（Type Safety）？

类型安全的本质就一句话：**编译器能在编译期发现类型错误，而不是等到运行时才崩。**

```java
// ✅ 类型安全 —— 编译期直接报错
List<String> names = new ArrayList<>();
names.add(123);   // 编译错误：int 不能放进 List<String>

// ❌ 类型不安全 —— 编译通过，运行时爆炸
List names = new ArrayList();          // raw type
names.add(123);                        // 编译通过
String s = (String) names.get(0);      // ClassCastException！
```

| 对比维度 | 类型安全 | 类型不安全 |
|---|---|---|
| **错误发现时机** | 编译期 | 运行时 |
| **表现** | IDE 红线、编译失败 | `ClassCastException` 崩溃 |
| **安全感** | 高，编译器帮你把关 | 低，埋了一颗定时炸弹 |

---

## 三、类型擦除为什么会导致类型不安全？

类型擦除本身不直接等于不安全——**擦除 + 绕过泛型检查**才是真正的杀手。以下几个场景覆盖了最常见的"绕过"方式：

### 1. Raw Type 混用（最常见 ⭐⭐⭐⭐⭐）

```java
List<String> names = new ArrayList<>();
List raw = names;          // raw type，绕过泛型检查
raw.add(123);              // 编译通过！没有泛型约束了
String s = names.get(0);   // ClassCastException 💥
```

发生了什么？

- 编译器以为 `names` 里全是 `String`
- 但 raw type 引用让 `Integer` 混了进去
- **运行时** `names` 就是普通的 `ArrayList`（`<String>` 被擦除了），JVM 看不出里面有"不应该存在"的东西
- 直到 `get(0)` 隐式强转失败，才崩

**Android 中常见场景：**

```java
// 场景1：老旧的第三方库返回 raw type
List data = SomeOldLib.getData();   // raw type 返回值
List<String> typed = data;          // 编译器只给警告，不报错
String first = typed.get(0);        // 💥 如果 data 里混入了其他类型

// 场景2：Fragment getArguments() 的泛型丢失
Bundle args = getArguments();
// 旧版 API（Android 13 起标记为 @Deprecated，但所有版本仍可调用）：
// 返回 Object，强转绕过泛型，运行时可能崩溃
List<String> items = (List<String>) args.getSerializable("items");
// 新版 API（Android 13+）：能校验容器类型，但元素泛型仍被擦除
// ArrayList items = args.getSerializable("items", ArrayList.class);
// 拿到的是 ArrayList，不是 List<String>，元素类型仍需自行校验

// 场景3：旧版 AsyncTask（已废弃但大量老项目仍在使用）
new AsyncTask<String, Void, List>() {  // 第三个泛型写成了 raw type
    @Override
    protected List doInBackground(String... params) {
        List result = new ArrayList();
        result.add("string");
        result.add(123);               // 💥 调用方以为得到了 List<String>
        return result;
    }
}.execute();
```

**如何避免：**
- 第三方库返回 raw type 时，对每个元素做 `instanceof` 检查后再强转
- 使用 `@SuppressWarnings("unchecked")` 时加注释说明为什么安全
- Java 项目开启 `-Xlint:unchecked`，把 unchecked 警告当错误处理

### 2. 反射绕过（⭐⭐⭐⭐）

```java
List<String> names = new ArrayList<>();
Method add = List.class.getMethod("add", Object.class);
add.invoke(names, 123);    // 反射完全绕过泛型
String s = names.get(0);   // ClassCastException 💥
```

反射操作的是**擦除后的字节码**，根本没有泛型概念，随便往里面塞东西。

**完整可运行的崩溃示例：**

> 需要添加 `kotlin-reflect` 依赖（`implementation("org.jetbrains.kotlin:kotlin-reflect")`）。

```kotlin
import kotlin.reflect.full.memberFunctions

fun main() {
    // 创建一个"看起来"类型安全的 List<String>
    val names: MutableList<String> = ArrayList()

    // 通过反射拿到擦除后的 add(Object) 方法
    val addMethod = ArrayList::class.memberFunctions
        .first { it.name == "add" && it.parameters.size == 2 }

    // 往 String 列表里塞 Integer
    addMethod.call(names, 42)      // ✅ 静默成功，JVM 毫无察觉
    addMethod.call(names, 3.14)    // ✅ 又塞了一个 Double

    // 在泛型方法中遍历
    names.forEach { name ->
        println(name.uppercase())  // 💥 java.lang.ClassCastException
        // Integer cannot be cast to String
    }
}
```

> 无论你用 Java 的 `Method.invoke` 还是 Kotlin 的 `KCallable.call`，结果都一样——JVM 字节码里没有泛型，反射看到的就是 `add(Object)`。

### 3. 泛型数组的协变漏洞（⭐⭐⭐）

先理解一个前置概念——**Java 数组是协变（covariant）的**：

```java
String[] strArr = new String[10];
Object[] objArr = strArr;  // ✅ 编译通过，String[] 是 Object[] 的子类型
objArr[0] = 123;           // ❌ 运行时 ArrayStoreException！
```

Java 数组保留运行时类型信息，能拦截类型不匹配的写入。**但泛型数组做不到这一点：**

```java
// 泛型数组——运行时类型被擦除，检查失灵
List<String>[] lists = (List<String>[]) new List<?>[10];
Object[] objArray = lists;
ArrayList<Integer> intList = new ArrayList<>();
intList.add(123);
objArray[0] = intList;                    // ✅ 静默成功！运行时 lists 就是 List[]
String s = lists[0].get(0);               // 💥 ClassCastException：Integer cannot be cast to String
```

`List<String>[]` 在运行时就是 `List[]`，任何 `List` 都能放进去——数组的运行时类型保护被擦除彻底破坏了。

#### Kotlin 的改善：`Array<T>` 是不可变（invariant）的

```kotlin
val strArr: Array<String> = arrayOf("a", "b")
val objArr: Array<Any> = strArr   // ❌ 编译错误！
// Type mismatch: inferred type is Array<String> but Array<Any> was expected
```

为什么 Kotlin 这样设计？

| | Java `String[]` | Kotlin `Array<String>` |
|---|---|---|
| **型变规则** | 协变：`String[]` 是 `Object[]` 的子类型 | 不可变：`Array<String>` 和 `Array<Any>` 没有子类型关系 |
| **编译期安全性** | 有漏洞，编译通过但运行时可能 `ArrayStoreException` | 完全安全，编译期就能阻止类型错误 |
| **泛型数组** | `List<String>[]` 编译警告，运行时可被污染 | 创建泛型数组需 unchecked cast，编译期无法保证安全 |

Kotlin 选择 invariant 就是为了**从根上堵住数组协变的安全漏洞**——既然 JVM 的数组协变会跟泛型擦除起冲突，那干脆不让你写出有不安全可能的代码。

### 4. 桥接方法（Bridge Method）的后门（⭐⭐）

```java
class Box<T> {
    T value;
    public void set(T v) { this.value = v; }
}

class StringBox extends Box<String> {
    @Override
    public void set(String v) { super.set(v); }
}
```

编译后，`StringBox` 实际有**两个** `set` 方法：

```text
void set(String v)    ← 你写的
void set(Object v)    ← 编译器生成的桥接方法，内部调 set((String) v)
```

桥接方法是为了保持多态——JVM 的方法分派基于方法签名（名称 + 参数类型），`Box.set(T)` 擦除后是 `set(Object)`，而 `StringBox.set(String)` 是另一个签名。编译器必须生成一个 `set(Object)` 作为桥接，把调用转发给 `set(String)`。

但如果通过反射调用桥接方法：

```java
StringBox stringBox = new StringBox();
Method bridge = StringBox.class.getMethod("set", Object.class);
try {
    bridge.invoke(stringBox, 123);  // 💥 InvocationTargetException，cause 是 ClassCastException
} catch (java.lang.reflect.InvocationTargetException e) {
    // e.getCause() → ClassCastException: Integer cannot be cast to String
}
```

> `getMethod` 只能获取 `public` 方法；如果 `set` 是包内可见的，需改用 `getDeclaredMethod("set", Object.class)` 并 `setAccessible(true)`。

这是擦除为多态制造的"后门"——正常使用不会有问题，但反射可以破坏这个约定。

---

## 四、四种场景对比

| 场景 | 绕过方式 | 崩溃时机 | 常见程度 | 防御措施 |
|---|---|---|---|---|
| **Raw Type 混用** | 无泛型引用操作泛型容器 | `get()` 时 ClassCastException | ⭐⭐⭐⭐⭐ | Kotlin 禁止 raw type；Java 开启 `-Xlint:unchecked` |
| **反射绕过** | `Method.invoke()` / `KCallable.call()` 直接操作擦除后的方法 | `get()` 或消费时 ClassCastException | ⭐⭐⭐⭐ | 避免反射操作泛型容器；使用 TypeToken 保留类型 |
| **泛型数组协变** | 数组运行时类型检查被擦除破坏 | `get()` 时 ClassCastException | ⭐⭐⭐ | Kotlin 禁止泛型数组；Java 优先使用 `List` 代替数组 |
| **桥接方法** | 反射调用编译器生成的 `set(Object)` 桥接 | `invoke()` 时 `InvocationTargetException`（cause 为 ClassCastException） | ⭐⭐ | 反射调用时指定精确方法签名而非 `Object` 版本 |

---

## 五、Kotlin 有没有改善？

Kotlin 做了不少努力，核心思路是**在编译器层面堵漏洞**，但 JVM 的限制决定了本质不会变：

### 5.1 `List<*>`：禁止 raw type 的替代方案

Kotlin 不允许你写 `List`（raw type），必须写 `List<*>`（star projection）：

```kotlin
val list: MutableList<Any> = mutableListOf("a", 1, true)   // ✅ 明确声明 Any
val anyList: MutableList<Any?> = mutableListOf("a", null)  // ✅ 可空

// List<*> 的语义："我并不知道元素类型，所以不让你乱写"
val starList: MutableList<*> = mutableListOf("a", "b")
starList.add("c")     // ❌ 编译错误！类型不匹配
starList.add(null)    // ❌ 编译错误！null 也不能写（因为 * 对应的可能是非空类型）

// 但读取是安全的——读出来至少是 Any?
val item: Any? = starList[0]   // ✅ 编译通过
```

`List<*>` 的使用场景与限制：

| 操作 | `MutableList<*>` | `MutableList<Any?>` | 说明 |
|---|---|---|---|
| **读取元素** | ✅ 返回 `Any?` | ✅ 返回 `Any?` | 两者都能读 |
| **写入非空值** | ❌ 编译错误 | ✅ 允许 | `*` 不知道目标类型，拒绝一切写入 |
| **写入 null** | ❌ 编译错误 | ✅ 允许 | 因为 `*` 投影出的类型可能是非空的 |
| **典型场景** | 只消费不生产的数据（如函数参数只想遍历） | 需要同时读写的异构容器 | — |

换句话说：**`List<*>` 用"禁止写入"换来了类型安全**。

### 5.2 `reified`：内联 + 类型实化

```kotlin
// ✅ reified 让 T 在运行时可用
inline fun <reified T> isType(value: Any): Boolean {
    return value is T   // 编译通过！因为函数体内联后 T 被替换成具体类型
}

// 使用
isType<String>("hello")  // 内联后等价于 "hello" is String → true
isType<Int>("hello")     // 内联后等价于 "hello" is Int    → false
```

**`reified` 的局限性：**

```kotlin
// ⚠️ 可以与类的泛型参数同名，但会遮蔽外部 T（编译器给出 warning）
class TypeChecker<T> {
    inline fun <reified T> check(value: Any): Boolean { ... }
    //                    ^ Warning: Type parameter 'T' shadows type parameter 'T' of containing class
}

// ✅ 换个名字更清晰——避免 shadowing 造成混淆
class TypeChecker2<T> {
    inline fun <reified R> check(value: Any): Boolean = value is R
}

// ❌ 只能在 inline 函数中使用
fun <reified T> normalCheck(value: Any): Boolean { ... }
//    ^ 编译错误！只有 inline 函数才能用 reified

// ❌ noinline 的 lambda 中不可用——lambda 作为普通函数对象传递，T 被擦除
inline fun <reified T> process(noinline block: (T) -> Unit) {
    // block 内部 T 已被擦除，无法使用 T::class
}

// ✅ 正确用法：结合 Gson 的 TypeToken 做反序列化
inline fun <reified T> Gson.fromJson(json: String): T {
    return this.fromJson(json, object : TypeToken<T>() {}.type)
    // reified 让 TypeToken 能捕获到 T 的具体类型
}

val user: User = gson.fromJson(json)  // 无需传 Class<User> 参数
```

**`reified` 的本质：** 它**不是**把泛型信息保留到了运行时，而是编译器把 `inline` 函数体**复制粘贴**到调用处，然后用具体类型替换 `T`。类的泛型参数无法被"内联"，所以 `reified` 只能用在 `inline` 函数上。

### 5.3 改善对比总结

| 改进点 | 原理 | 效果 | 局限性 |
|---|---|---|---|
| 禁止 raw type | 编译期强制写 `List<*>` | 堵住 raw type 漏洞 | 防不了反射 |
| `Array<T>` invariant | 拒绝 `Array<String>` → `Array<Any>` 的赋值 | 消灭数组协变漏洞 | 不解决泛型擦除根本问题 |
| `List<*>` star projection | 编译期禁止写入 | 只消费不生产的集合天然安全 | 需要写入时改用 `MutableList<Any?>` 明确声明异构意图 |
| `reified` 类型参数 | `inline` 函数体内联 + 类型替换 | `is T` / `T::class` 可用 | 仅限 inline 函数；不能与外层同名泛型参数冲突 |
| 声明处型变 (`out` / `in`) | 在类定义时声明 `List<out T>` / `Comparator<in T>` | 减少调用处的型变标注 | 不解决擦除问题，是编译期约束 |

**结论：** Kotlin 把最危险的 raw type 和数组协变漏洞堵上了，日常使用比 Java 安全得多。但底层的 JVM 字节码依然擦除，泛型反射、序列化反序列化等场景中的限制在 Kotlin/JVM 中依旧存在。**真正理解类型擦除，才能在这些边界场景里写出正确的代码。**

---

## 六、一句话总结

> **类型擦除**让泛型在运行时"消失"，**类型安全**靠编译期的泛型约束保证。一旦用 raw type、反射或数组协变绕过编译期检查，擦除就让 JVM 在运行时完全"看不见"类型错误——直到 `ClassCastException` 爆炸。

---

*延伸阅读：如果想深入了解 JVM 泛型擦除的实现细节，推荐阅读《Java 虚拟机规范》中关于 `Signature` 属性和 bridge method 的章节。*
