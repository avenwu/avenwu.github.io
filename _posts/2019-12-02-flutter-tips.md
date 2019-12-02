---
layout: post
title: "Flutter实用小技巧"
description: ""
header_image: /assets/img/2019-12-02-01.jpeg
keywords: ""
tags: [Flutter]
---
{% include JB/setup %}
![img](/assets/img/2019-12-02-01.jpeg)

* 目录
{:toc #markdown-toc}

使用Dart开发Flutter应用，和Java非常类似，因此对Dart语言特性和Flutter Framework积累足够的话，便可以写出更高效和代码。

分享几个实用的小技巧，本文参考了[FlutterDartTips](https://github.com/ibhavikmakwana/FlutterDartTips)，去除了一些很常见的写法。

## 发布模式判断
判断当前环境是否为发布模式。

```dart
const bool kReleaseMode = bool.fromEnvironment('dart.vm.product')
```
也可以使用`foundation`提供的常量，实现相同:
```dart
import 'package:flutter/foundation.dart';

print('Is Release Mode: $kReleaseMode');
```
使用这个可以用于控制日志输出，比如release模式关闭日志：
```dart
if (isProduction) {
  debugPrint = (String message, {int wrapWidth}) => {};
}
```

详情=》https://api.flutter.dev/flutter/foundation/kReleaseMode-constant.html

## 为Container设置背景图
都知道`Container`支持child设置展示内容，为了展示层叠效果，可以使用Column,其实还可以使用decoration间接实现背景图
```dart
Container(
  width: double.maxFinite,
  height: double.maxFinite,
  decoration: BoxDecoration(
    image: DecorationImage(
      image: NetworkImage('https://bit.ly/2oqNqj9'),
    ),
  ),
  child: Center(
    child: Text(
      'Flutter.dev',
      style: TextStyle(color: Colors.red),
    ),
  ),
),
```

## 断言提示
使用asert进行断言，通过第二个参数，提供个性化文案，可以让使用者对断言要求有一个更清楚的说明
```dart
assert(age > 18, "age should be >18");
```

## "链式"调用

利用Dart语法，可以简化方法调用
```dart
class Person {
  String name;
  int age;
  Person(this.name, this.age);
  void data() => print("$name is $age years old.");
}

void main() {
   // Without Cascade Notation
   Person person = Person("Richard", 50);
   person.data();
   person.age = 22;
   person.data();
   person.name += " Parker";
   person.data();
   
   // Cascade Notation with Object of Person
   Person("Jian", 21)
    ..data()
    ..age = 22
    ..data()
    ..name += " Yang"
    ..data();
}
```

## 空值处理
比较常见的一个判断，当一个变量为空时进行赋值操作。

```dart
// User below
title ??= "Title";

// instead of
if (title == null) {
  title = "Title";
}
```

## 参考
* https://github.com/ibhavikmakwana/FlutterDartTips
