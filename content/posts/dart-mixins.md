---
title: "Dart 面向对象：从类与继承到 mixin"
date: 2026-10-22T09:00:00+08:00
draft: false
tags: ["Dart", "Flutter", "Kotlin"]
categories: ["Flutter"]
---

学过 Kotlin，再看 Dart 的类、继承和抽象，大多能找到熟悉的概念。mixin 则需要单独认识。

这篇先用小例子串起这些基础，再重点理解 mixin。每个 Dart 代码块都是独立示例，可以分别放进 DartPad 运行。

![dart_mixins](https://raw.githubusercontent.com/visiongem/BlogImages/master/flutter/2026/dart_mixins.png)

## 1. 类、对象、变量和方法

**类定义对象有哪些数据、能做什么；对象是按这个定义创建的具体实例。**

```dart
class User {
  final String name; // 实例变量，也叫字段：每个用户各有一个名字

  User(this.name); // 构造函数：把传入的名字赋给 name

  void sayHello() { // 实例方法：通过对象调用，可以访问它的字段
    print('你好，我是 $name');
  }

  static bool isValidName(String name) { // 静态方法：通过类名调用
    return name.trim().isNotEmpty; // 去掉首尾空白后，名字是否非空
  }
}

void main() {
  final user = User('小明'); // 创建 User 对象，new 可以省略
  print(user.name); // 小明
  user.sayHello(); // 你好，我是 小明
  print(User.isValidName('  ')); // false，不需要先创建 User 对象
}
```

这里 `name` 是对象里的字段，`user` 是 `main()` 中保存对象引用的局部变量。

实例方法通过 `user.sayHello()` 调用；静态方法通过 `User.isValidName()` 调用，没有代表当前对象的 `this`。字段也可以用 `static` 声明，表示属于类，而非每个对象各保存一份。

对照 Kotlin：`User(this.name)` 的作用类似主构造函数中声明 `val name: String`。[Dart 官方说明：类与静态成员](https://dart.dev/language/classes)

Dart 的静态方法和 Kotlin `companion object` 中的方法，都可以通过类名调用。区别是：**Kotlin 的伴生对象本身是一个对象，其方法是对象的成员；Dart 的静态方法属于类，不需要对象。** 类里有伴生对象，也不代表这个类本身就是单例。[Kotlin 官方说明：伴生对象](https://kotlinlang.org/docs/object-declarations.html#companion-objects)

Dart 可以实现单例，只是没有 Kotlin `object AppConfig` 这样的专用声明语法。学完下面的构造函数，就能理解它的常见写法。

## 2. 构造函数：对象怎么创建、数据怎么初始化？

构造函数用于创建对象。下面把普通构造、命名构造和工厂构造放在一起比较：

```dart
class User {
  final String name;

  User(this.name); // 普通构造函数，直接用参数初始化字段

  // 命名构造函数；冒号后是初始化列表，在构造函数体执行前初始化字段。
  User.guest() : name = '游客';

  // 工厂构造函数：先处理数据，再决定返回什么对象。
  factory User.fromInput(String input) {
    final name = input.trim();
    if (name.isEmpty) return User.guest();
    return User(name);
  }
}

void main() {
  print(User('小明').name); // 小明
  print(User.guest().name); // 游客
  print(User.fromInput('  小红  ').name); // 小红
  print(User.fromInput('   ').name); // 游客
}
```

几个容易混淆的地方：

- `User(this.name)`：普通构造函数，不写返回类型。如果类没有声明任何构造函数，Dart 会提供一个无参的默认构造函数。
- `User.guest()`：命名构造函数，用名字区分创建用途。冒号后的 `name = '游客'` 是初始化列表，常用于初始化 `final` 字段。
- `factory User.fromInput(...)`：工厂构造函数，可以处理数据后返回当前类或其子类的对象，也可以返回已有对象。它不保证每次创建新对象，且不能访问 `this`。

这些概念可以组合：`User.fromInput` 既有名字，也是工厂构造函数；初始化列表则是初始化字段的写法，不是独立的构造函数类型。

Flutter 中还常见 `const` 构造函数，要求实例字段为 `final` 等。调用时显式加 `const`，或处于常量上下文中，且参数都是编译期常量，才会创建常量对象。仅在声明处加 `const`，不代表所有调用都是常量。[Dart 官方说明：构造函数](https://dart.dev/language/constructors)

### 用构造函数实现单例

单例的目的，是让使用者拿到同一个对象。Dart 常见的写法有下面两种。

**写法一：通过静态字段获取实例。**

```dart
class AppConfig {
  AppConfig._(); // 私有命名构造函数，其他库不能调用

  static final AppConfig instance = AppConfig._(); // 保存共享实例
  String theme = 'light';
}

void main() {
  final a = AppConfig.instance;
  final b = AppConfig.instance;
  a.theme = 'dark';
  print(b.theme); // dark，因为 a、b 指向同一个对象
  print(identical(a, b)); // true：判断是否为同一个对象
}
```

`static` 让你通过类名访问，`final` 防止 `instance` 指向另一个对象；对象里的 `theme` 仍然可以修改。`AppConfig._()` 是自己起名的私有构造函数，`_` 不是单例专用语法。

**写法二：用 factory 返回保存好的实例。**

```dart
class AppConfig {
  AppConfig._();

  static final AppConfig _instance = AppConfig._();
  factory AppConfig() => _instance; // 每次调用都返回同一个实例

  String theme = 'light';
}

void main() {
  final a = AppConfig(); // 看起来像创建对象，实际取回已有实例
  final b = AppConfig();
  print(identical(a, b)); // true
}
```

两种写法都保存一个实例，主要区别是调用入口：前者用 `AppConfig.instance`，后者用 `AppConfig()`。**factory 本身不等于单例，始终返回同一个对象才是这里实现单例的关键。** [Dart 官方说明：工厂构造函数](https://dart.dev/language/constructors#factory-constructors)

也可以通过静态 getter、静态方法提供入口，或者让依赖管理工具统一创建并复用实例。入门先掌握上面两种即可。注意，私有构造函数在同一个库内仍可调用，需要约定只在初始化共享实例时使用。

## 3. 封装、getter 和 setter

**封装就是把数据和操作放在一起，并通过约定的入口控制数据怎样被使用。** 例如音量只允许是 0～100：

```dart
class Player {
  int _volume = 50; // 下划线开头，表示库内私有

  int get volume => _volume; // getter：读取音量

  set volume(int value) { // setter：修改前检查范围
    if (value < 0 || value > 100) {
      throw ArgumentError('音量必须在 0～100 之间');
    }
    _volume = value;
  }
}

void main() {
  final player = Player();
  print(player.volume); // 50，读取时调用 getter，不加括号
  player.volume = 80;   // 赋值时调用 setter
  print(player.volume); // 80
  // player.volume = 200; // 取消注释后，会抛出 ArgumentError
}
```

这和 Kotlin 属性的自定义 `get()`、`set()` 类似。普通字段本来就有隐式 getter，可写字段还有 setter；需要校验或计算时，再显式编写即可。[Dart 官方说明：getter 与 setter](https://dart.dev/language/methods#getters-and-setters)

注意：Dart 的 `_volume` 是**库内私有**，同一文件中的其他类也能访问。通常一个 Dart 文件就是一个库，也可以通过 `part` 合并多个文件。其他库必须通过公开入口操作这里的音量。[Dart 官方说明：库与可见性](https://dart.dev/language/libraries)

## 4. 继承、多态、抽象：用一个例子理解

```dart
abstract class Animal { // 抽象类：这里不能直接创建 Animal 对象
  void speak(); // 抽象方法：只声明要求，把具体做法留给子类

  void sleep() => print('睡觉'); // 抽象类也可以有已经实现的方法
}

class Dog extends Animal { // 继承 Animal
  @override // 表示这里重写了父类成员，类似 Kotlin 的 override
  void speak() => print('汪汪');
}

class Cat extends Animal {
  @override
  void speak() => print('喵喵');
}

void main() {
  final animals = <Animal>[Dog(), Cat()]; // 用同一个父类型保存不同子类对象
  for (final animal in animals) {
    animal.speak(); // 依次输出“汪汪”“喵喵”：根据实际对象调用不同实现
  }
  animals.first.sleep(); // 睡觉：直接复用父类的方法
}
```

对照代码理解这几个词：

- **继承**：`Dog extends Animal`，Dog 可以复用 Animal 的成员，也可以重写它们。Dart 通过 `extends` 指定一个父类。
- **多态**：变量类型是 `Animal`，实际对象可以是 Dog 或 Cat。同样调用 `speak()`，会执行各自的实现。
- **抽象类**：用 `abstract class` 声明，可以包含字段、构造函数和已实现的方法，也可以声明抽象方法。
- **抽象方法**：如 `void speak();`，只有声明，没有方法体；可实例化的子类要提供实现，也可以从继承或混入中获得实现。

这些概念和 Kotlin 接近。不过，Dart 普通类和实例方法默认允许继承、重写，不需要先加 Kotlin 的 `open`。`final class` 可以禁止其他库继承或实现这个类，同一库内仍然允许。[Dart 官方说明：继承](https://dart.dev/language/extend)、[类修饰符](https://dart.dev/language/class-modifiers)

到这里，继承已经能复用父类代码了。如果两个不同种类的类只想复用某一项功能，就可以考虑下面的 mixin。

## 5. mixin：让不同的类复用一组功能

**mixin 用来把一组可复用的成员加入类中**。这组成员可以包含方法，也可以包含保存数据的字段。中文通常叫“混入”，mixins 就是它的复数形式。

用 `mixin` 定义，用 `with` 使用。

### 两个类想复用同一段功能

假设按钮和下载任务都需要记录重试次数，可以把计数功能写成一个 mixin：

```dart
mixin RetryCounter {
  int retryCount = 0; // 每个使用它的对象，都有自己的计数

  void recordRetry() {
    retryCount++;
  }
}

class DownloadTask with RetryCounter {}
class RetryButton with RetryCounter {}

void main() {
  final task = DownloadTask();
  task.recordRetry(); // 方法来自 RetryCounter
  task.recordRetry();
  print(task.retryCount); // 2

  final button = RetryButton();
  print(button.retryCount); // 0，和 task 的计数互不影响
}
```

读 `class DownloadTask with RetryCounter` 时，可以理解为：**给 DownloadTask 加上 RetryCounter 提供的计数功能**。

两个对象各自保存 `retryCount`，不会共用一个计数器。这个 mixin 只记录次数，发起重试的逻辑需要另写。

`mixin RetryCounter` 不能像普通类一样用 `RetryCounter()` 创建对象，也不能声明普通构造函数。你创建的是使用它的类的对象。[Dart 官方说明：Mixins](https://dart.dev/language/mixins)

### 已经继承了父类，还能用 mixin 吗？

可以。Dart 的 `extends` 只能指定一个父类，`with` 可以列出多个 mixin：

```dart
class Task {
  void start() => print('开始任务');
}

mixin Logging {
  void log(String message) => print('[日志] $message');
}

mixin RetryCounter {
  int retryCount = 0;
  void recordRetry() => retryCount++;
}

class DownloadTask extends Task with Logging, RetryCounter {}

void main() {
  final task = DownloadTask();
  task.start();             // 来自父类 Task
  task.log('下载失败');      // 来自 Logging
  task.recordRetry();      // 来自 RetryCounter
  print(task.retryCount);  // 1
}
```

`extends Task` 复用父类的实现，`with Logging, RetryCounter` 加入日志和计数功能。[Dart 官方说明：类](https://dart.dev/language/classes)

### 和 Kotlin 接口有什么关系？

Kotlin 的接口可以提供方法实现，这一点能帮助理解 mixin：

```kotlin
interface Logging {
    fun log(message: String) {
        println("[日志] $message")
    }
}

class DownloadTask : Logging

fun main() {
    DownloadTask().log("开始下载")
}
```

上面 Kotlin 的 `Logging` 接口，与前面 Dart 的 `Logging` mixin，都能让类复用 `log()` 的实现。

需要区分的是：**Dart mixin 可以定义保存状态的字段**，例如 `retryCount = 0`。Kotlin 接口可以声明属性或实现 getter，但不能有保存属性值的 backing field（幕后字段）。[Kotlin 官方说明：接口](https://kotlinlang.org/docs/interfaces.html)

另外，别把 Dart 的 `implements` 理解成 Kotlin 接口默认方法的复用方式：

| Dart 写法 | 作用 |
| --- | --- |
| `extends Task` | 继承一个父类，复用父类的实现 |
| `with Logging` | 混入功能，复用 mixin 的实现 |
| `implements Logging` | 遵守接口要求，不会获得 `log()` 的实现，需要自行提供 |

Dart 的 `implements` 不继承被实现类型的方法体。[Dart 官方说明：隐式接口](https://dart.dev/language/classes#implicit-interfaces)

### 多个 mixin 有同名方法怎么办？

对于签名兼容的同名方法，**后混入的实现会覆盖前面的实现**：

```dart
mixin A {
  void sayHello() => print('来自 A');
}

mixin B {
  void sayHello() => print('来自 B');
}

class Demo with A, B {}

void main() {
  Demo().sayHello(); // 来自 B；如果改成 with B, A，就输出“来自 A”
}
```

如果 `Demo` 自己也重写了 `sayHello()`，就优先调用 `Demo` 的实现。同名方法的参数、返回类型仍须符合重写规则，不能仅靠调整顺序解决类型冲突。

这和 Kotlin 多个接口默认方法冲突时，需要在实现类中显式重写的规则不同。

### on 是什么？先看懂使用限制

有些 mixin 需要调用父类的方法，可以用 `on` 声明对父类的要求：

```dart
class Task {
  void start() => print('开始任务');
}

mixin StartLogging on Task {
  @override
  void start() {
    print('记录启动日志');
    super.start(); // 调用已有的父类实现
  }
}

class DownloadTask extends Task with StartLogging {}

// 编译错误：普通 Object 不满足 on Task 的要求。
// class Button with StartLogging {}

void main() {
  DownloadTask().start(); // 依次输出“记录启动日志”“开始任务”
}
```

这里 `on Task` 的意思是：**混入 StartLogging 之前，类的继承链要满足 Task 类型的要求**。本例通过 `extends Task` 满足要求，因此 mixin 中可以使用 `super.start()`。

`on Task` 声明的是限制，不会替使用者自动继承 `Task`。普通功能复用通常不需要写 `on`。[Dart 官方说明：on 子句](https://dart.dev/language/mixins#use-the-on-clause-to-declare-a-superclass)

### 在 Flutter 里会在哪里遇到？

写动画时，常见这样的类声明：`class _DemoPageState extends State<DemoPage> with SingleTickerProviderStateMixin`。

把它分成两部分读：

- `extends State<DemoPage>`：继承页面的状态类。
- `with SingleTickerProviderStateMixin`：加入提供单个 `Ticker` 的能力。`Ticker` 会随动画帧触发回调，动画控制器靠它推进动画。

有了这个 mixin，创建 `AnimationController` 时就能传入 `vsync: this`，让当前 State 提供所需的 Ticker。

如果一个 State 的整个生命周期内需要创建多个 Ticker，就应使用 `TickerProviderStateMixin`。现在先读懂 `with` 的作用，动画的完整写法可以在学 Flutter 动画时再实践。[Flutter 官方说明：SingleTickerProviderStateMixin](https://api.flutter.dev/flutter/widgets/SingleTickerProviderStateMixin-mixin.html)

![dart_mixins_save](https://raw.githubusercontent.com/visiongem/BlogImages/master/flutter/2026/dart_mixins_save.png)

## 6. 实际写代码怎么选？

多个类需要复用一组方法，尤其这组方法还要维护各自的状态时，可以考虑 mixin。只是把字符串格式化、计算数值这类独立操作抽出来，普通函数通常就够了。

复习时，可以对照这些例子检查自己：构造函数负责创建对象；封装控制数据的使用入口；继承复用父类成员，多态让同一调用表现出不同的行为；抽象声明要求，mixin 复用一组功能。

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**
