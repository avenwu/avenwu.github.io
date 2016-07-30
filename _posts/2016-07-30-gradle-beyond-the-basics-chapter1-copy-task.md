---
layout: post
title: "Gradle进阶 第一章 Copy Task"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/gradle-beyond-the-basics-cover.png
keywords: "Gradle Beyond the Basics"
tags: [Gradle, Grade进阶]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/gradle-beyond-the-basics-cover.png)

## 前言

本文译自《Gradle Beyond the Basics》第一章 Copy Task.

译文仅作学习交流，喜欢的朋友请支持正版。

* 正版书籍：[http://shop.oreilly.com/product/0636920019923.do](http://shop.oreilly.com/product/0636920019923.do)
* 全书中文翻译：[http://gradle-beyond-the-basics.avenwu.net/](http://gradle-beyond-the-basics.avenwu.net/)

## 正文

拷贝任务是Gradle提供核心任务之一。在执行时，一个copy task会将原文件拷贝至目标目录，原文件可以来自一个或多个地方。由使用者决定从何处获取文件，拷贝至哪里，以及如果过滤这些文件。最简单的过滤配置如下所示

Example 1-1. 一个简单的拷贝task配置

```groovy
task copyPoems(type: Copy) {
  from 'text-files'
  into 'build/poems'
}
```

这个例子假设存在一个名为text-files的目录，包含一些文本诗歌。执行这个脚本可以通过gradle copyPoems，会把那些诗歌文件拷贝至build/poems目录下，便于后续加工操作。

默认情况下，from指定的目录下所有的文件都包含在这个copy动作中。我们可以通过显式指定include或exclude规则来改变这个默认行为。inclu和exclude的规则遵循Ant风格，`**`表示递归匹配任何目录名称，`*`表示匹配文件名中的任意字符。include配置默认会把其他未指定的文件全部视为exclude,相似的exclude配置默认也会把未指定的文件视为include状态。

当exclude和include一起使用时，Gradle优先匹配exclude。也就是先将所有include的文件会中，然后根据exclude配置，移除所有相关的文件。

如果无法在一个语句中完整表示include/exclude的配置，那么可以多写几个；
Example 1-2. 拷贝task，拷贝除了Henley之外的所有诗歌

```groovy
task copyPoems(type: Copy) {
  from 'text-files'
  into 'build/poems'
  exclude '**/*henley*'
}
```
Example 1-3. 只拷贝Shakespare和Shelley的诗歌

```groovy
task copyPoems(type: Copy) {
  from 'text-files'
  into 'build/poems'
  include '**/sh*.txt'
}
```
通常构建操作就是把一些文件从各处聚合在一个地方。要实现这个操作，只需要用多个from来include，如Example 1-4。每个from配置都可以有自己的一套include/exclude配置。

Example 1-4. 从多处拷贝文件的task

```groovy
task complexCopy(type: Copy) {
  from('src/main/templates') {
    include '**/*.gtpl'
  }
  from('i18n')
  from('config') {
    exclude 'Development*.groovy'
  }
  into 'build/resources'
}
```
