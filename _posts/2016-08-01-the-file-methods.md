---
layout: post
title: "Gradle基础进阶 文件相关方法"
description: "Gradle基础进阶 第一章 文件相关方法"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-07-31-01.png
keywords: "Gradle Beyond the Basics"
tags: [Gradle]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-07-31-01.png)

## 前言

本文译自《Gradle Beyond the Basics》Chapter 1. The File Methods.

译文仅作学习交流，喜欢的朋友请支持正版。

* 正版书籍：[http://shop.oreilly.com/product/0636920019923.do](http://shop.oreilly.com/product/0636920019923.do)
* 全书中文翻译：[http://gradle-beyond-the-basics.avenwu.net/](http://gradle-beyond-the-basics.avenwu.net/)

## 正文

在Gradle构建中有许多文件相关的方法可供选择。这些方法是Project对象的方法，也就是说可以在构建中的任何位置调用这些方法。并且提供了一些方法将路径转换为project的相对路径用java.io.File表示，创建文件集合，递归处理目录树。后续我们将逐一学习这些内容。

## file()

file()方法用于创建一个java.io.File对象，将project的相对路径转换为一个绝对路径。file()方法接受一个参数，可以是一个字符串，一个file对象，一个URL，或者返回值为上述三者之一的一个closure代码块。

当一个task有一个File参数时，file()会很有用。比如说，Java插件提供的一个task叫jar，这个task会创建一个JAR文件，默认包含sourceSet's中的类和resource资源。构建好的JAR文件会放在build目录下的一个默认位置，但是有的构建任务可能需要改变这个默认位置。jar task有一个属性叫做destinationDir,可以用来改变位置，可能出现如下写法，Example 1-12：

*Example 1-12 尝试修改jar任务的文件目标位置*

```groovy
jar {
  destinationDir = 'build/jar'
}
```

但是上述构建执行的时候会失败，原因是destinationDir是File类型而不是一个String，既然这样，那就改成File如Example 1-13：

*Example 1-13 试修改jar任务的文件目标位置*

```groovy
jar {
  destinationDir = new File('build/jar')
}

```

这一层成功build了，但是并不是所有情况下都符合预期。File的构造器会在我们提供的参数之外创建一个绝对路径，这个路径是相对于JVM的启动路径。这就导致这个目录会随着我们启动JVM的方式而变化，比如我们是直接启动Gradle，还是通过wrapper，亦或通过IDE面板，以及持续构建的server服务器。解决这个问题的办法是使用file()方法，如Example 1-14.

*Example 1-14. 用file()方法修改destinationDir位置*

```groovy 
jar {
  destinationDir = file('build/jar')
}
```

通过上述设置，destinationDir被成功修改为project根目录下的build/jar目录。这是最符合预期的行为，并且不会由于Gradle启动方式不同而发生变化。

如果已经有了一个File对象，file()方法会尝试将他转换为相对project的路径。new File('build/jar')没有显式指定父目录，因此file(new File('build/jar'))的父目录就是当前的project根目录。这个例子不是太合适--实际代码中我们通常会避免直接进行这种内部File构建，不过file的操作效果和签名的示例是一致的。某些情况下如果你已经有了File对象，可以这么操作。

file()也支持java.net.URL和java.net.URI对象，只要他们的协议是file://。File URLs并不常见，但是在用ClassLoader加载资源时经常出现。如果你正好在构建中遇到了file URL,那么就可以直接用file()转换为project相对路径了。

## files()

files()方法根据提供的参数返回一个文件集合，和file()类似，不过他操作的是文件集合，files()接受许多不同参数，具体图Table 1-1

*Table 1-1. files()接受的参数*

| 参数类型 | 方法表现行为 |
| -------- | -------- |
| String | 创建一个集合，包含单独的一个文件，如题file一样解析文件名 |
| java.io.File | 创建一个集合，包含单独的一个文件，如题file一样解析文件对象 |
| java.net.URL或者java.net.URI | 根据file创建集合，只支持file:// URLs |
| Collection, Iterable,Array | 创建一个文件集合，包含所有的参数文件，集合元素是递归解析的因此他们可能包含其他files()所允许的类型 |
| Task | 根据task输出创建文件结合，每个task都会定义输出文件集合：files(compileJava) |
| Task 输出 | 表现和task名称一样，不过允许TaskOutputs对象被显示命名 |

这如你所看到的，files()在创建文件集合这方面方法非常多，可以接受不同的参数。

Gralde初学者经常以为files的返回值是传统的集成了List接口的集合类。但实际上files返回的是FileCollection,这个即可是Gradle中文件编程的基础接口。在文件集合章节我们更详细介绍。

## fileTree()

file()有效的将路径转为文件，files()基于此创建文件集合。但是如果你想要遍历目录树，收集找到的所有文件，然后把它们作为集合对待该如何实现呢？通过fileTree()可以很好的完成这个任务。

有三种方法唤起fileTree()，每一种都高度借鉴了copy task的配置。它们都有一些特性，其共同点是：方法必须制定遍历的根目录，可选的配置include和exclude。

使用fileTree()的最简单方式是指定一个父目录，运行它遍历所有的子目录，把找到的文件都放入文件集合。例如创建一个集合包含Java Project中所有的源代码，就可以用fileTree(‘src/main/java’)完成。

有时候可能需要做一些简单的过滤操作。比如你知道源码目录中包含了一些以~结尾的备份文件。又比如源码目录目录中混入了一些XML文件，而你需要单独处理这些XML文件。这种情况下课通过fileTree()操作，Example 1-15.

*Example 1-15. fileTree()结合includes和excludes*

```groovy
def noBackups = fileTree('src/main/java') { 
    exclude '**/*~'
}

def xmlFilesOnly = fileTree('src/main/java') { 
    include '**/*.xml'
}
```

目录和include，exclude都可以用map形式传递，如Example 1-16.

*Example 1-16. 利用map结合fileTree与include，exclude*

```groovy
def noBackups = fileTree(dir: 'src/main/java', excludes: ['**/*~']) 
def xmlFilesOnly = fileTree(dir: 'src/main/java', includes: ['**/*.xml'])
```