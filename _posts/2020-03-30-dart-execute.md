---
layout: post
title: "Dart构建产物与使用"
description: "在Flutter开发中，几乎所有的入门介绍都会提到他的多种编译产物和技术细节"
header_image: /assets/img/2020-03-30-01.png
keywords: ""
tags: [Flutter]
---
{% include JB/setup %}
![img](/assets/img/2020-03-30-01.png)

* 目录
{:toc #markdown-toc}

## 背景
在Flutter开发中，几乎所有的入门介绍都会提到他的多种编译产物和技术细节。但是，这些其实并不是Flutter社区的首创，也不是他的核心优势。诸如AOT和JIT的概念在其他的编程语言也都或多或少存在，在语言上的优势，本身得益于Dart。本文主要根据实际问题处理，来介绍一下Dart的几种编译产物。

* Dart有哪些编译产物
* 如何编译并运行这些产物
* 适用场景

## 编译产物
Dart VM可以识别多种编译产物，一般来说我们通过编译模式实现更快的运行速度和开发效率。在Dart 2.7之后，引入了dart2native，开发者可以实现本地机器码的编译，产物是可运行的格式，内部会嵌套完整的Dart运行环境，从而解除使用者的Dart环境依赖，同时提高运行速度。美中不足的是还不支持交叉编译。

根据Dart源码转变为Dart VM能识别的产物的时机，我们可以将产物划分为以下几种情况：

* dart源码
* dart snapshot，这一块又可以细分为JIT和AOT
* native机器码

举例来说，我们有一个最简单源文件hello.dart。正式业务下，一般会有工程组织，引入pubspec.yaml管理依赖。

```dart
// hello.dart
main() => print('Hello World');
```

通过源码直接运行
```
aven$ dart hello.dart 
Hello World
```
## Kernel Snapshots
通过snapshot模式运行，首先需要生成Kernel Snapshot

>dart --snapshot-kind=kernel --snapshot=hello.dart.snapshot hello.dart

得到的快照比源文件体积大一些，运行时启动时间更短，一个snapshot是多个dart源文件合并的产物。
```
aven$ ls -al
-rw-r--r--   1 aven  staff   37 Mar 30 11:48 hello.dart
-rw-r--r--   1 aven  staff  408 Mar 30 11:52 hello.dart.snapshot
```
快照本身是一种Kernel AST形式的二进制文件，Kernel AST指的是一种Dart VM能够识别的一种二进制中间产物，可以近似的理解为是Java的Class文件。他是架构无关的，因此可以在平台之间迁移使用。

运行的时候执行dart即可，如果有参数可以在快照文件之后加空格跟上。

> dart hello.dart.snapshot

## JIT Application Snapshots (app-jit)
将快照类型改为`app-jit`编译后即可得到JIT的Snapshot

> dart --snapshot-kind=app-jit --snapshot=hello.dart.jit.snapshot hello.dart

我们再次看下产物大小，
```
aven$ ls -al
-rw-r--r--   1 aven  staff       37 Mar 30 11:48 hello.dart
-rw-r--r--   1 aven  staff  3704160 Mar 30 12:22 hello.dart.jit.snapshot
-rw-r--r--   1 aven  staff      408 Mar 30 11:52 hello.dart.snapshot
```
可以发现JIT的快照比存Kernel快照要大很多。这主要是因为其中包含了解析后的类，编译生成的代码
> all the parsed classes and compiled code generated during a training run of program

运行是类似的，只需要通过dart执行
> dart hello.dart.jit.snapshot

## AOT Application Snapshots (app-aot)
AOT模式在2.6版本需要通过dart2aot编译，通过dartaotruntime运行

>$ dart2aot hello.dart hello.dart.aot
$ dartaotruntime hello.dart.aot
Hello World

在2.7的时候dart2aot被dart2native替代了。

> dart2native hello.dart -k aot

默认aot产物后缀为aot，名字相同，也可以通过-o 指定产物的路径和名称。
```
aven$ ls -al
-rw-r--r--   1 aven  staff  1185000 Mar 30 12:59 hello.aot
-rw-r--r--   1 aven  staff       37 Mar 30 11:48 hello.dart
-rw-r--r--   1 aven  staff  3704160 Mar 30 12:22 hello.dart.jit.snapshot
-rw-r--r--   1 aven  staff      408 Mar 30 11:52 hello.dart.snapshot
```

在相同hello.dart源码构建后，aot的产物大小小于jit。

## Native Machine Code机器码
最后还可以通过dart2native产物平台相关的机器码。
> dart2native hello.dart

默认产物为exe后缀。

```
aven$ ls -al
-rw-r--r--   1 aven  staff  1185000 Mar 30 12:59 hello.aot
-rw-r--r--   1 aven  staff       37 Mar 30 11:48 hello.dart
-rw-r--r--   1 aven  staff  3704160 Mar 30 12:22 hello.dart.jit.snapshot
-rw-r--r--   1 aven  staff      408 Mar 30 11:52 hello.dart.snapshot
-rwxr-xr-x   1 aven  staff  5104888 Mar 30 13:04 hello.exe
```

四种产物大小关系如下：
> Dart Source Code < Kernel Snapshot < AOT Snapshot < JIT Snapshot < Native Machine Code

![](/assets/images/dart-compile-output.png)

## 模式选用
根据我们已经掌握的编译和产物特征，就可以适当选择编译方式了。

> 假如你在验证一个bug，或者测试一段代码的兼容情况

通过编写Dart代码，并生成Kernel Snapshot，然后可以提供给多人验证，比如Windows平台，Linux，MacOS等。

> 假如你要交付一个可执行文件给用户

通过源码生成机器码，这样既可以保证执行速度，有没有无需环境依赖，在生成的机器码中，自带了精简版的Dart运行环境。

## 小结
属性dart的产物有助于我们更好的在不同场景选用编译方式。

* [https://dart.dev/tools/dart2native](https://dart.dev/tools/dart2native)
* [https://github.com/dart-lang/sdk/wiki/Snapshots](https://github.com/dart-lang/sdk/wiki/Snapshots)
* [https://github.com/dart-lang/sdk/tree/master/pkg/kernel](https://github.com/dart-lang/sdk/tree/master/pkg/kernel)
