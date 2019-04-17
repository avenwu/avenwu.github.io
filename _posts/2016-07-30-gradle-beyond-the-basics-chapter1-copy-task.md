---
layout: post
title: "Gradle基础进阶 拷贝任务"
description: ""
header_image: /assets/img/2016-07-31-01.png
keywords: "Gradle Beyond the Basics"
tags: [Gradle]
---
{% include JB/setup %}
![img](/assets/img/2016-07-31-01.png)

## 前言

本文译自《Gradle Beyond the Basics》Chapter 1. Copy Task.

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

## 改变目录结构

Example 1-4的输出结果是吧所有源文件集中到了一个扁平的目录build/resources下面。当然你可能不希望所有文件都放到这个扁平的目录下，比如你可能想保持一些目录结构或者映射到新的目录树。要达到这个效果，可以简单地配置一下from的代码块，如Example 1-5

Example 1-5 拷贝task，将源目录映射为新的目录

```groovy
task complexCopy(type: Copy) {
  from('src/main/templates') {
    include '**/*.gtpl'
    into 'templates'
  }
  from('i18n')
  from('config') {
    exclude 'Development*.groovy'
    into 'config'
  }
  into 'build/resources'
}
```

注意看配置变化，顶层的into配置任然是需要的，否则的话build文件执行不会成功。内层的每个from中的into是相对于外层into配置的。

加入拷贝的文件数或者文件大小很大的话，那么这个拷贝操作在一次完整的过程中，时间开销会比较昂贵。Gradle的增量编译特性可以减轻这个问题。在编译过程中，gradle首次完整编译耗时长，随后的增量编译时间相对较短。

## 拷贝并且重命名文件

如果编译中需要拷贝文件，那么很多情况下也会需要重命名文件。文件名可能会需要标示开发环境，符合特定环境下的标准，亦或是根据产品特殊配置。不管具体是什么原因，Gradle提供了两种方式来完成重命名工作：正则表达式和Groovy closures。

用正则来重命名的话，只需要提供一个源文件的正则表达式和目标文件名。这个源文件的正则表达式表示符合条件的文件，通过group提取文件的部分名称，最终的目标文件形如$1/$2。比如把一些配置文件拷贝到工作区目录，Example 1-6:

Example 1-6 通过正则重命名

```groovy
task rename(type: Copy) {
  from 'source'
  into 'dest'
  rename(/file-template-(\d+)/, 'production-file-$1.txt')
}
```

为了动态编码重命名，我们可以利用closure给rename方法(Example 1-7),这个代码块接收一个文件名参数，代表源文件名称，返回值是重命名后的名称。

Example 1-7 动态编码重命名

```groovy
task rename(type: Copy) {
  from 'source'
  into 'dest'
  rename { fileName ->
    "production-file${(fileName - 'file-template')}"
  } 
}
```
