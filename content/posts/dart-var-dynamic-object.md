---
title: "Dart 中 dynamic、var、Object 的区别"
date: 2026-10-21T12:00:00+08:00
draft: false
tags: ["Dart", "Flutter", "Kotlin"]
categories: ["Flutter"]
---

学 Dart 时，下面三种写法都能保存字符串，很容易觉得它们差不多：

```dart
var a = 'Dart';
dynamic b = 'Dart';
Object c = 'Dart';
```

区别主要在两点：**后续能赋什么类型的值，以及调用成员时是否进行静态检查**。静态检查，就是代码运行前，编译器先检查用法是否符合类型要求。

先记住三者的区别：

- `var`：让编译器推断类型。
- `Object`：能保存各种非空对象，使用时保留类型检查。
- `dynamic`：允许动态调用成员，调用是否有效要到运行时确认。

以下示例彼此独立，普通语句放进 `main()` 中运行；错误写法已注释。

![dart_type_封面图](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/flutter/2026/dart_type_封面图.png)

## 1. var：让编译器推断类型

```dart
var name = 'Dart'; // 根据初始值，推断 name 的类型是 String

name = 'Flutter';          // 可以，仍然是字符串
print(name.toUpperCase()); // FLUTTER，转成大写

// name = 123; // 编译错误：不能把 int 赋给 String 变量
```

这里的 `var name = 'Dart'`，效果相当于 `String name = 'Dart'`。

`var` **是声明变量的关键字，不是类型**。和 Kotlin 的 `var name = "Dart"` 一样，推断为 `String` 后，就不能再赋值为整数。

不过，Dart 的 `var` 有一种容易忽略的情况：

```dart
var value; // 没有类型注解，也没有初始值，类型是 dynamic
value = 'Dart'; // 这次赋值不会把变量类型固定为 String
value = 123;   // 可以，变量的类型仍是 dynamic
print(value); // 123
```

如果局部变量暂时没有初始值，可以明确写类型，例如 `String value;`，并在读取前赋值。[官方说明：变量与类型推断](https://dart.dev/language/variables)

## 2. dynamic：调用成员时，把检查留到运行时

```dart
void main() {
  dynamic value = 'Dart';
  print(value.toUpperCase()); // DART，当前值是字符串，调用成功

  value = 123; // 可以换成整数
  print(value); // 123

  // 取消下一行注释后，运行到这里会抛出 NoSuchMethodError。
  // 编译器允许这样写，但 int 没有 toUpperCase() 方法。
  // print(value.toUpperCase());
}
```

`dynamic` 可以接收任意类型的值，包括 `null`。调用成员时，编译器不检查对象是否有这个成员，运行时才确认。**所以，编辑器不报错，不代表代码能正常运行**。

## 3. Object：能接收不同类型，但仍然检查用法

`Object` 是 Dart 中所有非空对象的共同父类型。字符串、数字、自定义类的对象，都可以赋给 `Object` 变量。

```dart
Object value = 'Dart';
print(value); // Dart

value = 123;  // 可以，int 也是 Object
print(value); // 123

// value = null; // 编译错误：Object 不接受 null
```

区别在于调用方法时是否检查类型：

```dart
void showValue(Object value) {
  print(value.toString()); // 可以，Object 本身就有 toString()

  // print(value.toUpperCase()); // 编译错误：Object 没有这个方法

  if (value is String) {
    // 确认是 String 后，编译器允许使用字符串的方法。
    print(value.toUpperCase());
  }
}

void main() {
  showValue('Dart'); // 依次输出 Dart、DART
  showValue(123);    // 只输出 123
}
```

这里的参数声明为 `Object`，编译器还不知道它是不是字符串，因此不能直接调用字符串专有的方法。`is String` 判断成功后，就可以了，这叫**类型提升**，类似 Kotlin 的智能转换。

`Object` 和 `dynamic` 都能保存不同类型的对象，区别是 `Object` **保留静态类型检查**。Dart 官方也建议：只需要接收各种对象时，优先考虑 `Object` 或 `Object?`。[Effective Dart：dynamic 的使用建议](https://dart.dev/effective-dart/design#avoid-using-dynamic-unless-you-want-to-disable-static-checking)

## 4. Object?：在 Object 的基础上允许 null

这里的 `?` 和 Kotlin 一样，表示允许为 `null`：

```dart
Object? value = null; // 可以，也能赋值为字符串、整数等
print(value); // null
```

`Object?` 仍保留类型检查。使用字符串专有的方法前，可以用 `is String` 判断，它也会排除 `null`。

对照着记就行：

| Dart | Kotlin 中可用于理解的对应概念 |
| --- | --- |
| `var name = 'Dart'` | `var name = "Dart"`，根据初始值推断类型 |
| `Object` | `Any`，可以接收各种非空对象 |
| `Object?` | `Any?`，也可以接收 `null` |
| `value is String` 后使用字符串方法 | `is String` 后的智能转换 |
| `dynamic` | 普通 Kotlin／Android 代码中没有直接对应项，不能当成 `Any` 理解 |

以上是概念对照。[Kotlin 官方说明：类型检查与智能转换](https://kotlinlang.org/docs/typecasts.html)

## 5. 为什么用了 var，还是可能出现运行时类型错误？

使用 `var` 时，**推断出的类型也可能是 dynamic**。除了前面的 `var value;`，解析 JSON 时也常遇到：

```dart
import 'dart:convert';

void main() {
  var data = jsonDecode('{"name":"小明"}'); // jsonDecode 返回 dynamic
  print(data['name']); // 小明；这里 data 的类型也是 dynamic
}
```

如果想保留类型检查，可以用 `Object?` 接住，再确认结构和字段：

```dart
import 'dart:convert';

void main() {
  Object? data = jsonDecode('{"name":"小明"}');

  if (data is Map<String, dynamic>) {
    Object? name = data['name']; // 字段可能缺失，也可能不是字符串
    if (name is String) {
      print(name); // 小明
    }
  }
}
```

`Object?` 不改变数据，只约束你怎样使用它。这里检查了结果的类型；JSON 文本本身格式错误，仍会抛出异常。[jsonDecode 官方说明](https://api.dart.dev/dart-convert/jsonDecode.html)

另一个常见情况是 `var items = [];`：这里没有足够的信息推断元素类型，所以得到 `List<dynamic>`。如果只想保存字符串，可以写成 `var items = <String>[];`。[官方说明：Dart 类型系统](https://dart.dev/language/type-system)

## 6. 实际写代码怎么选？

| 需求 | 写法 |
| --- | --- |
| 从初始值就能看出类型 | `var name = 'Dart';` |
| 想明确标注类型 | `String name = 'Dart';` |
| 接收不同类型的非空对象，并保留类型检查 | `Object` |
| 还需要接收 `null` | `Object?` |
| 确实需要动态调用，并接受可能的运行时错误 | `dynamic` |

不需要重新赋值时，用 `final`，例如 `final name = 'Dart';`。它类似 Kotlin 的 `val`，限制的是重新赋值；`final data = jsonDecode(...)` 的类型仍然是 `dynamic`。

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**
