---
layout: post
title: "Gradle基础进阶 过滤与修改文件"
description: "Gradle基础进阶 第一章 过滤与修改文件"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-07-31-01.png
keywords: "Gradle Beyond the Basics"
tags: [Gradle]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-07-31-01.png)

## 前言

本文译自《Gradle Beyond the Basics》Chapter 1. Filtering and Transforming Files.

译文仅作学习交流，喜欢的朋友请支持正版。

* 正版书籍：[http://shop.oreilly.com/product/0636920019923.do](http://shop.oreilly.com/product/0636920019923.do)
* 全书中文翻译：[http://gradle-beyond-the-basics.avenwu.net/](http://gradle-beyond-the-basics.avenwu.net/)

## 正文

通常构建任务要做的不仅仅是拷贝、重命名文件，也会需要改变这些被拷贝文件的内容。Gradle提供了三种核心方式来处理这个需求，它们是：expand()方法，filter()方法，以及eachFIle()方法.。接下来我们逐一来看看这些方法

## 关键词Expansion

在一个常见的构建用例中，会把一些配置文件拷贝到工作空间，然后在拷贝过程中替换一些文件内容。其中可能有一个特殊的配置文件，他包含了一系列的参数，这些参数是不随着开发环境变化的，而是通过增补一小部分参数来区分。在这种情况下，由于该配置文件从工作目录拷贝到构建目录，如果能同时替换作为变量的字符串参数，那么会方便的很多。expand()方法就是Gradle提供的，用来解决此问题。

expand()方法依赖于Groovy的SimpleTempleEngine类。SimpleTemplateEngine添加了一个替代性的语法关键词，他和Groovy中字符串插入比较类似。在{}大括号中的$变量表示需要替代的部分。在copy task中声明expansion关键词的时候，需要传入一个map，map中的key将会匹配并替换模板文件中{}内对应的同名变量。
所谓**千言万字总无情，总是代码留人心**，来看过具体案例：

Example 1-8 通过expansion关键词拷贝文件

```groovy
task copyProductionConfig(type: Copy) {
  from 'source'
  include 'config.properties'
  into 'build/war/WEB-INF/config'
  expand([
    databaseHostname: 'db.company.com',
    version: versionId,
    buildNumber: (int)(Math.random() * 1000), 
    date: new Date()
  ]) 
}
```

> SimpleTemplateEngine也提供了其他的一些特性可以在expand()中使用，详细的细节可以前往[在线文档](http://docs.groovy-lang.org/latest/html/api/groovy/text/SimpleTemplateEngine.html)查询

特别要注意的是传给expand()方法的是Groovy中的map，它是由中括号括起来的键值对，key与value直接用冒号分割，每个键值对之间用都好分开。在上面的案例中，task的expand配置将在配置文件拷贝时替换map内的数据到文件内。实际工作中的构建任务，不一定会这么实现，也可能有其他的操作，map中的数据可能根据环境而采用不同值。最终传给expand的map数据可以来自任何地方。由于Gradle的构建脚本是可执行的Groovy代码实现，在这几乎提供了无限的操作可能。

针对上面的案例我们一起看一看拷贝前后的文件变化，加深理解：

拷贝的源文件：

```
    #
    # Application configuration file
    #
    hostname: ${databaseHostname}
    appVersion: ${version}
    locale: en_us
    initialConnections: 10
    transferThrottle: 5400
    queueTimeout: 30000
    buildNumber: ${buildNumber}
    buildDate: ${date.format("yyyyMMdd'T'HHmmssZ")}
```

拷贝后的目标文件：

```
    #
    # Application configuration file
    #
    hostname: db.company.com
    appVersion: 1.6
    locale: en_us
    initialConnections: 10
    transferThrottle: 5400
    queueTimeout: 30000
    buildNumber: 77
    buildDate 20120105T162959-0700
```

## 逐行过滤

expand()方法完美适用于常规的文本替换，甚至可以做一些轻量级的复杂操作。但是有些情况下我们需要逐行独立操作文件内容。这个时候filer()就派送用场了。

filter()方法有两种形式：一种是接受closur代码段，一种是接受class类。这两种我们都会接受，现在先看一下closure形式的。

当传递一个closure给filter()方法后，这块代码会在每一行内容过滤的时候执行。在closure中执行任何想做的操作，然后返回过滤后的行内容。举个例子，将Markdown文本转换为HTML,我么利用MarkdownJ来实现以下，Example 1-9.

Example 1-9. 通过closure执行filter过滤文本

```groovy
import com.petebevin.markdown.MarkdownProcessor
buildscript {
  repositories {
    mavenRepo url: 'http://scala-tools.org/repo-releases'
  }
  dependencies {
    classpath 'org.markdownj:markdownj:0.3.0-1.0.2b4'
  } 
}
task markdown(type: Copy) {
  def markdownProcessor = new MarkdownProcessor() into 'build/poems'
  from 'source'
  include 'todo.md'
  rename { it - '.md' + '.html'}
  filter { line ->
    markdownProcessor.markdown(line)
  }
}
```

被处理的markdown内容是一小段诗歌，包含一些注释说明：

```
    # A poem by William Carlos Williams
    so much depends
    upon
    # He wrote free verse
    a red wheel
    barrow
    # In the imageist tradition
    glazed with rain
    water
    # And liked chickens
    beside the white
    chickens
```

拷贝过滤之后，把注释都换成空白行：

```
    so much depends
    upon
    a red wheel
    barrow
    glazed with rain
    water
    beside the white
    chickens
```

Gradle提供了灵活性很高的逐行处理的逻辑空间，这也是因为Gradle在设计上希望能把复杂的过滤逻辑和task定义剥离开。所以与其把过滤的逻辑都耦合在task上，可以选择用clas的方式吧过滤的具体逻辑拆独立出去，这也使得自动的模块测试更简单。

现在我们不传递closue给filter()方法，取而代之的是一个定义好的class，这个class是java.io.FilterReader的实例。Ant API提供了包含丰富接口的FilterReader实现，这个在Gradle当中也建议复用。Example 1-9 以新的方式重写后得到Example 1-10

Example 1-10. 自定义的Ant Filter类实现文本过滤拷贝

```groovy
import org.apache.tools.ant.filters.*
import com.petebevin.markdown.MarkdownProcessor
buildscript {
  repositories {
    mavenRepo url: 'http://scala-tools.org/repo-releases'
  }
  dependencies {
    classpath 'org.markdownj:markdownj:0.3.0-1.0.2b4'
  } 
}
class MarkdownFilter extends FilterReader { 
    MarkdownFilter(Reader input) {
    super(new StringReader(new MarkdownProcessor().markdown(input.text))) 
    }
}
task copyPoem(type: Copy) {
  into 'build/poems'
  from 'source'
  include 'todo.md'
  rename { it - ~/\.md$/ + '.html'}
  filter MarkdownFilter
}
```

最后这个MarkdownFilter也可以完整的移出build文件本身，相关细节会在plug-in章节介绍。

## 遍历文件过滤

expand()和filter()方法提供了基于所有文件的拷贝过程中的文件操作，还有一种情况我们需要把每个文件当中一个独立个体来考虑，这个时候可以让eachFile()上场。

eachFile()可以接受一个closure，在每个文件处理是被执行。这个closure有一个参数，代表当前的FileCopydetails实例。FileCopyDetails允许我们把整个文件的内容作为一个整体考虑。他提供了一些有用的方法，比如重命名文件，改变目标地址，exclude文件，在其他路径创建备份以及以java.io。File的形式提供动态的文件处理。你可以Gradle DSL的形式来做很多事，也可以直接操控。举例来说，假设你需要拷贝一些文件，然后在拷贝过程汇总要计算这些文件的SHA1的哈希值，并把这些哈希值存到另外一个文件。

Example 1-11. 通过eachFile计算文件的哈希值

```groovy
import java.security.MessageDigest; 
task copyAndHash(type: Copy) {
  MessageDigest sha1 = MessageDigest.getInstance("SHA-1");
  into 'build/deploy'
  from 'source'
  eachFile { fileCopyDetails ->
    sha1.digest(fileCopyDetails.file.bytes)
  }
doLast {
    Formatter hexHash = new Formatter() 
    sha1.digest().each { b -> hexHash.format('%02x', b) } 
    println hexHash
  } 
}
```