---
title: "一篇学完 Dart 基础：Flutter 入门前的 12 个关键知识点"
date: 2026-10-20T12:00:00+08:00
draft: false
tags: ["Flutter", "Dart"]
categories: ["Flutter"]
---

学习 Flutter，先掌握 Dart 的变量、函数、集合、类和异步编程，就能读懂大部分入门代码。本文分为 12 个部分，用关键语法和示例讲清楚这些知识。

![flutter1_封面图](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/flutter/2026/flutter1_封面图.png)

## 1. 什么是 Dart

Dart 是 Flutter 使用的编程语言，支持类型推断、面向对象和异步编程。开发阶段可使用 JIT（即时编译），发布到原生平台时可使用 AOT（提前编译）；Web 则有相应的 Web 编译目标。

程序从 `main()` 开始执行：

```dart
void main() {
  print('Hello, Dart!');
}
```

`void` 表示不返回有用的值，语句通常以分号结束。单行注释用 `//`，多行注释用 `/* ... */`，文档注释用 `///`。

### 先用 DartPad 动手：不用安装开发环境

[DartPad](https://dartpad.dev/) 是 Dart 官方提供的在线编辑器，打开浏览器就能编写和运行 Dart，适合边读本文边练习。国内访问不顺畅时，可以尝试 [dartpad.cn](https://dartpad.cn/)。

使用步骤：

1. 打开网站，选择 Dart 示例或创建 Dart 代码。
2. 在编辑区输入带有 `main()` 的完整程序。
3. 点击 **Run** 运行，在输出区查看 `print()` 的结果；有错误时，根据编辑器提示修改。

可以先粘贴下面这段代码，再修改列表里的数字，观察输出变化：

```dart
void main() {
  final scores = [60, 75, 90];
  final passed = scores.where((score) => score >= 70).toList();
  print('达到 70 分的成绩：$passed');
}
```

DartPad 也支持 Flutter 示例，可以在浏览器里预览简单界面。学习本文时先用 Dart 模式练习语法，之后再切换到 Flutter 示例，观察 Widget 如何组成页面。

注意，DartPad 是浏览器中的练习工具：不支持 `dart:io` 这类依赖本地运行环境的库，也不能任意安装 pub.dev 上的第三方包，只能使用它支持的包。本文中的基础示例大多可以直接运行；本文代码块是彼此独立的示例，不要全部拼进同一个文件。只有语句的片段应放进 `main()`；只有函数或类定义的片段，需要添加 `main()` 调用；带 `await` 的片段应放进异步函数。网络请求还取决于包支持情况和目标接口的浏览器跨域限制。

以上能力与限制可查阅 [DartPad 官方说明](https://dart.dev/tools/dartpad)。需要完整项目、任意依赖或本地文件操作时，再安装 SDK，用 `dart run 文件名.dart` 运行。

## 2. 基础语法：变量、常量和运算符

```dart
var name = '小明';       // 推断为 String，之后不能赋值为整数
int age = 18;
double height = 1.75;
bool isStudent = true;
String greeting = '你好，$name，今年 ${age + 1} 岁';

final createdAt = DateTime.now(); // 运行时确定，只能赋值一次
const pi = 3.14159;               // 编译时常量
```

| 写法 | 关键含义 |
| --- | --- |
| `var` | 让编译器推断类型，变量仍有确定的类型 |
| `final` | 变量只能赋值一次，适合默认使用 |
| `const` | 编译时就能确定的常量 |
| `late` | 延迟初始化，必须在读取前完成赋值 |

`final` 限制的是变量重新赋值，不代表对象内容不能变化：

```dart
final names = ['小明'];
names.add('小红'); // 可以修改列表内容
// names = [];    // 错误：不能重新赋值

const fixedNames = ['小明'];
// fixedNames.add('小红'); // 运行时出错：常量列表不可修改
```

常用运算符包括 `+ - * /`、`== != > <`、`&& || !`。其中 `/` 得到浮点结果，`~/` 是截断除法，`%` 求余数：

```dart
print(5 / 2);  // 2.5
print(5 ~/ 2); // 2
print(5 % 2);  // 1
```

## 3. 类型与空安全：对照 Kotlin 看

Dart 常用类型有 `int`、`double`、`String`、`bool`。和 Kotlin 一样，**`String` 不能为 null，`String?` 可以为 null**。

空安全先记这张表就够了：

| 含义 | Kotlin | Dart |
| --- | --- | --- |
| 可以为 null | `String?` | `String?` |
| 不为空才访问 | `name?.length` | `name?.length` |
| 为空就用默认值 | `name ?: "游客"` | `name ?? '游客'` |
| 强行断言非空 | `name!!.length` | `name!.length` |

例如，“有名字就取长度，没有就用 0”：

```kotlin
// Kotlin
val length = name?.length ?: 0
```

```dart
// Dart，放在 main() 中运行
String? name;
final length = name?.length ?? 0;
print(length); // 0
```

Dart 还有一个 `??=`：`name ??= '游客';` 表示 **name 为 null 时，才给它赋值**。`??` 只是取默认值，不会修改 name。

平时优先用 `?.` 和 `??`。`!` 和 Kotlin 的 `!!` 一样，实际为 null 就会报错。这里的 null 不包括空字符串 `''`。

## 4. 函数：位置参数、命名参数和回调

```dart
int add(int a, int b) {
  return a + b;
}

int square(int n) => n * n; // 箭头函数只包含一个表达式
```

可选位置参数写在 `[]` 中；命名参数写在 `{}` 中，调用时带参数名：

```dart
String greet(String name, [String suffix = '!']) => '$name$suffix';

void createUser({required String name, int age = 18}) {
  print('$name: $age');
}

void main() {
  print(greet('小明'));
  createUser(name: '小红', age: 20);
}
```

`required` 表示必须传参，不代表参数一定不可空，是否可空仍由类型决定。

函数也可以作为参数传递，**就是 Kotlin 中的函数类型参数**。例如，传入一个函数，在任务完成时调用它：

```kotlin
// Kotlin
fun runTask(onDone: () -> Unit) {
    println("执行任务")
    onDone()
}

fun main() {
    runTask {
        println("完成")
    }
}
```

对应的 Dart 写法：

```dart
void runTask(void Function() onDone) {
  print('执行任务');
  onDone();
}

void main() {
  runTask(() {
    print('完成');
  });
}
```

两段代码都会依次输出“执行任务”“完成”。Dart 的 `() { ... }` 是匿名函数，对应上面 Kotlin 传入的 lambda。

| 函数类型 | Kotlin | Dart |
| --- | --- | --- |
| 无参数、无返回值 | `() -> Unit` | `void Function()` |
| 接收整数、无返回值 | `(Int) -> Unit` | `void Function(int)` |
| 接收整数、返回字符串 | `(Int) -> String` | `String Function(int)` |

读 `void Function() onDone` 时，可以拆成两部分：`void Function()` 是参数的类型，`onDone` 是参数名。Dart 把返回类型写在 `Function` 前，输入参数类型写在括号里。

**`onDone` 表示函数本身，`onDone()` 才是调用它。** “回调”说的是用途：你把函数交给另一段代码，由它在合适的时候调用。回调不一定是异步的，上面的例子就会立即执行。

## 5. 列表与集合：List、Set

`List` 有顺序、允许重复、下标从 0 开始；`Set` 用于保存不重复的元素。

```dart
final scores = <int>[80, 90, 90];
scores.add(100);
print(scores[0]); // 80
print(scores.length); // 4
scores.remove(80); // 删除第一个匹配值；按下标删除用 removeAt

final uniqueScores = scores.toSet();
print(uniqueScores); // {90, 100}

final Set<String> tags = {}; // 空 Set 要写类型
final emptyMap = {};         // 默认是空 Map，不是 Set
```

集合转换最常见的两个操作：

```dart
final numbers = [1, 2, 3, 4];
final doubled = numbers.map((n) => n * 2).toList();
final even = numbers.where((n) => n.isEven).toList();
print(doubled); // [2, 4, 6, 8]
print(even);    // [2, 4]
```

`map` 转换元素，`where` 筛选元素；它们返回 `Iterable`，需要列表时调用 `toList()`。

这里的 `isEven` 是 Dart 整数自带的属性，用来判断是不是偶数；`isOdd` 判断是不是奇数，结果都是 `bool`。**它们是属性，使用时不加 `()`。**

```dart
print(4.isEven); // true：4 是偶数
print(3.isEven); // false：3 不是偶数
print(3.isOdd);  // true：3 是奇数
```

所以 `where((n) => n.isEven)` 就是“保留所有偶数”，相当于 Kotlin 的 `filter { it % 2 == 0 }`。

展开运算符和集合中的 `if`、`for` 在 Flutter 中很常见：

```dart
final loggedIn = true;
final items = [
  '首页',
  if (loggedIn) '个人中心',
  ...['设置', '帮助'],
  for (var i = 1; i <= 3; i++) '栏目 $i',
];
```

## 6. 程序控制流

```dart
void main() {
  final score = 85;
  if (score >= 90) {
    print('优秀');
  } else if (score >= 60) {
    print('及格');
  } else {
    print('不及格');
  }

  final result = score >= 60 ? '通过' : '未通过';
  print(result);

  for (var i = 0; i < 3; i++) {
    print(i);
  }

  for (final name in ['小明', '小红']) {
    print(name);
  }

  var count = 0;
  while (count < 3) {
    count++;
  }
}
```

`break` 结束循环，`continue` 跳过本次循环。多分支也可以用 `switch`。Dart 3 的 switch 表达式适合把一个值映射成另一个值：

```dart
String statusText(int code) => switch (code) {
  200 => '成功',
  404 => '不存在',
  _ => '其他状态',
};
```

## 7. 映射 Map：键值对

```dart
final ages = <String, int>{
  '小明': 18,
  '小红': 20,
};

ages['小李'] = 22; // 新增或更新
print(ages['小明']); // 18
print(ages['不存在']); // null
print(ages.containsKey('小红')); // true
ages.remove('小李');

for (final entry in ages.entries) {
  print('${entry.key}: ${entry.value}');
}
```

即使声明为 `Map<String, int>`，下标读取结果仍是 `int?`，因为键可能不存在：

```dart
final age = ages['未知用户'] ?? 0;
```

`keys` 获取全部键，`values` 获取全部值，`entries` 用于同时遍历键和值。JSON 对象解析后通常也通过 Map 访问。

![flutter1_map](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/flutter/2026/flutter1_map.png)

## 8. 类与对象

类描述数据和行为，对象是类的实例。Dart 创建对象通常不写 `new`。

```dart
class User {
  final String name;
  int age;

  User({required this.name, this.age = 18});

  // 命名构造函数：提供另一种创建方式
  User.guest() : name = '游客', age = 0;

  bool get isAdult => age >= 18;

  void introduce() {
    print('我是 $name，今年 $age 岁');
  }
}

void main() {
  final user = User(name: '小明', age: 20);
  user.introduce();
  print(user.isAdult);

  final guest = User.guest();
  guest.age = 18;
}
```

`this.name` 是构造函数给字段赋值的简写；`get` 定义像属性一样读取的计算结果；`static` 定义通过类名访问的成员。

名称以 `_` 开头表示库内私有，并非仅限当前类。一个 Dart 文件通常构成一个库。

Flutter 中常见的 `const` 构造函数，允许创建常量对象。例如，创建一个坐标为 `(0, 0)` 的点：

```dart
class Point {
  final double x; // final：赋值后不能修改
  final double y;

  // this.x、this.y：将传入的参数直接赋给字段
  const Point(this.x, this.y);
}

void main() {
  const origin = Point(0, 0); // 创建常量对象，x、y 都是 0

  print(origin.x); // 0.0
  print(origin.y); // 0.0
}
```

构造函数上的 `const` 表示“允许创建常量对象”，要求实例字段为 `final`；`const origin` 表示“这里创建常量对象”，传入的值必须在编译时确定。

左边已有 `const`，右边可以省略，因此 `const origin = Point(0, 0);` 等价于 `const origin = const Point(0, 0);`。

## 9. 继承与方法重写

`extends` 继承父类，`@override` 标注重写，`super` 访问父类成员或调用父类构造函数。

```dart
class Animal {
  final String name;
  Animal(this.name);

  void speak() => print('发出声音');
}

class Dog extends Animal {
  Dog(String name) : super(name);

  @override
  void speak() => print('$name：汪汪');
}

void main() {
  final Animal animal = Dog('小白');
  animal.speak(); // 小白：汪汪
}
```

变量声明为父类类型，调用的仍然是实际对象重写的方法，这就是多态。

另外记住三个关键词：

- `abstract class`：抽象类，可声明没有实现的方法，不能直接实例化。
- `implements`：实现接口，需要实现接口要求的实例成员，不会继承实现。
- `with`：把 mixin 中已有的方法加入类中，直接复用。

**mixin 可以先理解为“一组可复用的方法”**，用 `mixin` 定义，用 `with` 加到类里：

```dart
mixin Fly {
  void fly() => print('飞行');
}

mixin Swim {
  void swim() => print('游泳');
}

class Duck with Fly, Swim {} // 同时获得 fly() 和 swim()

void main() {
  final duck = Duck();
  duck.fly();  // 飞行
  duck.swim(); // 游泳
}
```

可以类比 Kotlin 接口中的默认方法：已有实现可以直接复用。Dart 的 `implements` 不继承实现，`with` 则用于复用 mixin 的实现。

`extends` 后只能写一个父类，`with` 后可以写多个 mixin，用逗号隔开。两者可以一起使用：继承父类的同时，再通过 mixin 添加方法。

## 10. 泛型：先留一个类型的位置，使用时再确定

你已经见过泛型：`List<String>` 表示“元素是字符串的列表”，`List<int>` 表示“元素是整数的列表”。和 Kotlin 的 `List<String>`、`List<Int>` 是同一个思路。

自己定义类时，可以用 `T` 先代替具体类型。例如，一个盒子既可以用来装字符串，也可以用来装整数：

```dart
class Box<T> { // T 是类型的占位符，使用时再指定
  final T value; // 盒子里保存的值，类型是 T
  Box(this.value);

  T getValue() => value; // 取出的值也保持 T 类型
}

void main() {
  final textBox = Box<String>('Dart'); // T 是 String
  final numberBox = Box<int>(42);     // T 是 int

  print(textBox.getValue());   // Dart，返回类型是 String
  print(numberBox.getValue()); // 42，返回类型是 int

  // Box<int>('Dart'); // 编译错误：指定装 int，就不能传字符串
}
```

**`T` 不是某种固定类型，只是一个名字。** 上面复用的是同一份 Box 代码，但每个盒子仍有明确的类型。

再补两个常用点：

- **类型可以自动推断**：`Box('Dart')` 通常就能推断为 `Box<String>`，不必每次手写 `<String>`。
- **函数也能使用泛型**：例如取列表的第一个元素，传入字符串列表就返回字符串，传入整数列表就返回整数。

```dart
T first<T>(List<T> items) => items.first; // 接收 List<T>，返回 T

void main() {
  final name = first(['Dart', 'Flutter']); // String，值为 Dart
  final number = first([10, 20]);          // int，值为 10
  print(name);
  print(number);
}
```

这里的 `first` 要求列表非空，空列表会抛异常。泛型负责保证类型，是否为空仍需要另外处理。

## 11. 异步编程：Future、async、await

比如请求用户昵称，服务器需要过一会儿才能返回。异步函数会先返回一个 `Future<String>`，表示“这次操作成功完成后，会得到一个字符串”。**它不是昵称本身，而是代表这次操作结果的对象。** 操作也可能失败。

结合 Kotlin 协程，可以这样理解：

| Dart | Kotlin 协程中的类比 |
| --- | --- |
| `Future<String>` | 类似 `Deferred<String>`：代表稍后完成的结果 |
| `await task` | 类似 `task.await()`：等待并取出结果 |
| `await Future<void>.delayed(...)` | 类似 `delay(...)`：等待一段时间，不阻塞线程 |

**两边的 `async` 用法不同**：Dart 的 `async` 写在函数体前，标记异步函数，允许在函数体中使用 `await`；Kotlin 的 `async { ... }` 用于启动协程并返回 Deferred。

### 不加 await 拿到 Future，加上 await 拿到结果

```dart
Future<String> loadName() async {
  // 模拟请求耗时 1 秒，等待期间不会阻塞当前 isolate
  await Future<void>.delayed(const Duration(seconds: 1));
  return '小明'; // 返回 String，async 函数会通过 Future 交付它
}

Future<void> main() async {
  print('开始加载');

  final task = loadName(); // 发起操作，拿到 Future<String>
  print('昵称还没拿到，先做其他事');

  final name = await task; // 等结果，拿到的 name 是 String
  print('昵称：$name');
}
```

输出顺序是：先打印“开始加载”和“昵称还没拿到，先做其他事”，约 1 秒后打印“昵称：小明”。

对应的 Kotlin 协程代码如下（需要 `kotlinx.coroutines`）：

```kotlin
import kotlinx.coroutines.*

suspend fun loadName(): String {
    delay(1000)
    return "小明"
}

suspend fun main() = coroutineScope {
    println("开始加载")

    val task = async { loadName() } // Deferred<String>
    println("昵称还没拿到，先做其他事")

    val name = task.await() // String
    println("昵称：$name")
}
```

注意调用方式：Dart 的 `loadName()` 返回 Future；Kotlin 直接调用 `suspend` 函数 `loadName()` 会挂起等待并返回 String，要得到 Deferred 才使用 `async { loadName() }`。

如果没有其他事要先做，可以合成一句：

```dart
final name = await loadName(); // 放在 async 函数中
```

**`await` 可以类比协程的挂起等待**：当前函数后面的代码先暂停，结果准备好后再继续；等待期间，程序仍能处理其他事件。

不过，`async` 和 `await` **不会自动创建新线程**。把大量同步计算放进 async 函数，仍可能造成卡顿。

这个类比用于理解等待流程。Dart 的 Future 本身没有 Kotlin Job 那样的取消和父子协程关系。

### 再认识两个相关用法

多个互不依赖的请求，可以先一起发起，再等它们全部完成：

```dart
// 沿用上面的 loadName()，放在 async 函数中
final names = await Future.wait([loadName(), loadName()]);
print(names); // [小明, 小明]，两个操作一起开始，约 1 秒后得到结果
```

`Future.wait` 这里的作用是“等这一组结果”，不会把这些任务自动变成多线程执行。

`Future` 对应一次完成的结果；`Stream` 对应陆续到来的多个事件，比如一条条聊天消息。读取 Stream 可以用 `await for`，后面用到时再深入。

请求失败时，可以用 `try/catch` 包住带 `await` 的调用来处理错误，下一节有完整示例。

## 12. 获取网络数据：请求 → JSON → 对象

在 Dart 项目中先执行 `dart pub add http`；Flutter 项目使用 `flutter pub add http`。下面的完整示例请求一条文章数据，将 JSON 转换为有明确类型的对象：

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;

class Post {
  final int id;
  final String title;

  const Post({required this.id, required this.title});

  factory Post.fromJson(Map<String, dynamic> json) {
    final id = json['id'];
    final title = json['title'];
    if (id is! int || title is! String) { // is!：判断不是某种类型
      throw const FormatException('文章数据格式不正确');
    }
    return Post(id: id, title: title);
  }
}

Future<Post> fetchPost() async {
  final uri = Uri.https('jsonplaceholder.typicode.com', '/posts/1');
  final response = await http.get(uri).timeout(
    const Duration(seconds: 10), // 超过 10 秒未完成，就抛出超时异常
  );

  if (response.statusCode != 200) {
    throw Exception('请求失败：${response.statusCode}');
  }

  // 先把响应字节解码为文本，再解析 JSON
  final Object? data = jsonDecode(utf8.decode(response.bodyBytes));
  if (data is! Map<String, dynamic>) {
    throw const FormatException('预期返回 JSON 对象');
  }
  return Post.fromJson(data);
}

Future<void> main() async {
  try {
    final post = await fetchPost();
    print('${post.id}: ${post.title}');
  } catch (e) {
    print('加载文章失败：$e');
  }
}
```

这里串起了前面的知识：`Future<Post>` 表示异步返回文章，`await` 等待响应，`jsonDecode` 将 JSON 文本解析为 Dart 值，`factory` 构造函数负责创建对象，类型检查负责验证外部数据，`try/catch` 处理失败。

JSON 对象对应 Map，JSON 数组对应 List。实际业务应根据接口约定处理字段缺失、null 和类型变化，不能假设请求成功就一定能解析。

![flutter1_all](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/flutter/2026/flutter1_all.png)

## 面试延伸：把前面的基础再想深一步

下面的问题都从前面的变量、空安全、类、集合和异步编程展开。先理解“为什么”，不必背答案；涉及 Flutter 页面的问题先了解场景即可。

### 1. final 和 const 都不能重新赋值，有什么区别？

对应第 2 节“变量”和第 8 节“类与对象”。

**final 是“只能赋值一次”，const 是“编译时就确定的常量”。** 比如 `DateTime.now()` 要运行时才知道结果，可以赋给 final 变量，不能赋给 const 变量。final 也不会自动让列表内容变得不可修改。

构造函数写了 const，只表示允许创建常量对象。看下面两种调用：

```dart
class Point {
  final int x;
  const Point(this.x);
}

void main() {
  const a = Point(1);
  const b = Point(1); // 相同的常量对象会复用
  final c = Point(1); // 普通调用，创建另一个对象

  // identical 判断是不是同一个对象
  print(identical(a, b)); // true
  print(identical(a, c)); // false
}
```

Flutter 中复用 const Widget，可以减少父级重建时的一些更新工作，但不代表它永远不会重建。

### 2. String?、late 和 !，应该怎么选？

对应第 3 节“空安全”。它们表达的是三件不同的事：

| 写法 | 意思 | 例子 |
| --- | --- | --- |
| `String? name` | 允许没有值 | 用户可能没填昵称 |
| `late String name` | 稍后赋值，但读取前必须准备好 | 对象创建后，再进行初始化 |
| `name!` | 我保证此刻不是 null | 类似 Kotlin 的 `name!!` |

**业务允许没有值，就用 `?` 并处理 null；只是赋值时间晚，才考虑 late。** 没初始化就读取 late 变量会报错；实际为 null 却使用 `!`，同样会报错。

`late String name;` 可以类比 Kotlin 的 `lateinit var name: String`。Dart 还允许写 `late final`，表示稍后赋值、且只能赋值一次。

### 3. 写了 async，耗时操作就不会卡顿了吗？

对应第 11 节“异步编程”。

**不会。等待结果和执行计算是两回事。** `await` 可以让当前函数暂停等待，但 async 函数里的大量同步计算仍会占用当前执行环境。和 Kotlin 一样，异步写法不等于自动切换到后台线程。

```dart
Future<void> task() async {
  print('B');
  await Future<void>.delayed(Duration.zero);
  print('D');
}

void main() {
  print('A');
  task();
  print('C');
}
// A → B → C → D
```

先输出 A，调用 task 后立即输出 B；遇到 await，task 暂停，main 继续输出 C；等待结束后，task 才继续输出 D。即使等待时间是零，也不是立刻跳过 await 执行后面的代码。

`Future(() => heavyWork())` 只是把计算安排到稍后执行，不会自动换线程。大量计算可以考虑交给另一个 **isolate（拥有独立内存和事件循环的执行环境）**。原生平台可用 `Isolate.run`；Flutter Web 的处理方式需要另行考虑。

### 4. 都是“稍后执行”，为什么顺序不同？

这是第 11 节“异步执行顺序”的进一步延伸。Dart 会把一些稍后执行的任务放进队列，等待轮到它们：

- **微任务队列**：例如 `scheduleMicrotask` 添加的任务。
- **事件队列**：例如下面用 `Future(() { ... })` 安排的任务。

先执行完当前同步代码，再清空微任务队列，之后才处理下一个事件任务：

```dart
import 'dart:async';

void main() {
  print('1');
  Future<void>(() => print('事件'));
  scheduleMicrotask(() => print('微任务'));
  print('2');
}
// 1 → 2 → 微任务 → 事件
```

**微任务不会打断正在执行的同步代码。** 如果不停添加微任务，事件任务就可能一直等不到执行机会。这里演示的是两种具体 API，不能据此认为所有 Future 都以同一种方式排队。

![flutter1_task](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/flutter/2026/flutter1_task.png)

### 5. 为什么写了 try/catch，还是没接住异步错误？

对应第 11、12 节“异步与网络请求”。常见原因是：**只调用了异步函数，没有 await 它的结果。**

```dart
Future<void> save() async {
  throw Exception('保存失败');
}

Future<void> main() async {
  try {
    await save(); // 等待结果，失败时进入下面的 catch
  } catch (e) {
    print(e);
  }
}
```

如果把这里的 `await save()` 改成 `save()`，这个 try/catch 就接不住该 Future 后续传出的错误。

另外，**不再等待，不等于任务被取消**：`timeout()` 只限制等待时间；`Future.wait` 中一个任务失败，也不会自动取消其他任务。

Stream 的监听也需要管理。`listen` 返回一个订阅对象，相当于“继续接收消息”的凭证；页面不再需要这份监听时，应调用订阅的 `cancel()`。

### 6. 两个对象的字段值相同，就相等吗？

对应第 5、7、8 节“集合与对象”。

**默认不一定相等。** 普通类没有重写 `==` 时，比较的是不是同一个对象，不会自动逐个比较字段。想让两个用户编号只要数值相同就算相等，可以这样写：

```dart
class UserId {
  final int value;
  const UserId(this.value);

  @override
  bool operator ==(Object other) =>
      identical(this, other) || other is UserId && other.value == value;

  @override
  int get hashCode => value.hashCode; // 与上面的相等规则保持一致
}

void main() {
  print(UserId(1) == UserId(1)); // true
  print({UserId(1), UserId(1)}.length); // 1：Set 将它们视为重复元素
}
```

上面的比较规则是：同一个对象，或者都是 UserId 且 value 相同，就返回 true。

`hashCode` 是哈希值，常见的 Set、Map 会用它帮助查找元素。**相等对象必须有相同的 hashCode，但 hashCode 相同不代表对象相等。** 所以重写 `==` 时也要重写 `hashCode`。参与比较的字段使用 final，可以避免放进集合后字段变化、导致查找异常。

同样，普通 List 的 `==` 默认也不会逐个比较元素。

### 7. factory 构造函数一定创建新对象吗？

对应第 8 节“构造函数”和第 12 节“JSON 转对象”。

**不一定，取决于函数里返回什么。** factory 可以创建并返回新对象，也可以返回以前保存的对象。它不会自动帮你实现单例。

本文的 `Post.fromJson` 是“检查 JSON 字段，再调用 `Post(...)` 创建对象”。换成返回一个已保存的 Post，就能复用已有对象。

`fromJson` 只是我们取的名字，不是 Dart 关键字；`factory` 才是关键字，factory 函数体中不能使用 `this`。

### 8. 请求返回成功，为什么页面还会出错？

对应第 12 节“获取网络数据”，再向实际页面开发延伸一步：

- **数据格式不对**：状态码是 200，但 id 变成了字符串，仍然无法按预期解析。所以要检查字段类型。
- **旧结果覆盖新结果**：用户先搜 A，再搜 B，A 却更晚返回。可以给请求编号，只接受最新一次请求的结果。
- **用户已经离开页面**：结果回来时，页面可能已销毁。Flutter 中等待结束后，调用 `setState` 前检查 `mounted`；使用 BuildContext 前检查 `context.mounted`，确认对应对象仍挂在组件树上。
- **同一个操作提交多次**：按钮点击后可以暂时禁用；下单等关键操作，还要让服务端保证同一次操作不会被重复执行。

这一题先理解场景即可，`setState` 和页面生命周期会在学习 Flutter 时展开。

## 学完后，用这几个点检查自己

- 能区分 `var`、`final`、`const`，知道 `final` 本身不会让 List 内容变得不可修改。
- 能处理可空值，知道 `?`、`?.`、`??`、`!` 各自的用途。
- 能写命名参数、回调函数，以及集合的转换与筛选。
- 能用类描述数据，用构造函数创建对象，用重写实现不同的行为。
- 能读懂 `List<T>`、`Map<K, V>`、`Future<T>`。
- 能用 `async/await` 发请求，检查状态、解析 JSON，并处理异常。

可以用“获取文章列表并筛选标题”作为练习：定义 `Post` 类，请求列表接口，把 JSON 数组转换成 `List<Post>`，再用 `where` 筛选并打印标题。完成这个流程，就把 Dart 入门的主要知识串起来了。

## 参考资料

- [构造函数与 factory](https://dart.dev/language/constructors)
- [Dart 异步 API](https://api.dart.dev/dart-async/)
- [Kotlin 协程：async 与 Deferred](https://kotlinlang.org/docs/composing-suspending-functions.html)
- [Flutter 并发与 isolate](https://docs.flutter.dev/perf/isolates)
- [Dart 语言入门与各语法专题](https://dart.dev/language)
- [Dart 概览：类型与编译目标](https://dart.dev/overview)
- [Effective Dart：集合与函数的使用建议](https://dart.dev/effective-dart/usage)
- [异步编程：Future、async、await](https://dart.dev/libraries/async/async-await)
- [Dart 并发与 isolate](https://dart.dev/language/concurrency)
- [http 包 API 文档](https://pub.dev/documentation/http/latest/)

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**
