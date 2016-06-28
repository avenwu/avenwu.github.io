---
layout: post
title: "混淆实操--手把手教你用applymapping"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img//2016-06-21-01.jpg
keywords: ""
tags: []
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img//2016-06-21-01.jpg)

## 前言
在混淆代码中有一个-applymapping配置，主要是用来维持两次混淆公用一份mapping，确保相同的代码混淆后是一样的命名；

最近正好有朋友问到这个具体的实现，索性写个demo看看；

## 上手实操
大致思路其实就是上面提到的，先看一下官方对-applymapping的定义：

**-applymapping filename**

Specifies to reuse the given name mapping that was printed out in a previous obfuscation run of ProGuard. Classes and class members that are listed in the mapping file receive the names specified along with them. Classes and class members that are not mentioned receive new names. The mapping may refer to input classes as well as library classes. This option can be useful for incremental obfuscation, i.e. processing add-ons or small patches to an existing piece of code. If the structure of the code changes fundamentally, ProGuard may print out warnings that applying a mapping is causing conflicts. You may be able to reduce this risk by specifying the option -useuniqueclassmembernames in both obfuscation runs. Only a single mapping file is allowed. Only applicable when obfuscating.


对应到工程中就是在混淆配置中多加一句，如：

```
-applymapping mapping-v1.0.txt
```
就一句话，本文实际就可以结束了，当然为了更具说服力，下面我们写一个案例来演示一下这个完整的过程。

## 演示工程
创建一个演示用的工程,本文的工程放在这里

[https://github.com/avenwu/applymappingdemo](https://github.com/avenwu/applymappingdemo)

现在添加一个测试用的类，HelloMonic.java，然后打包生成apk，观察得到的mapping.tx和apk

### mapping.txt
内容很多，我们直接去关键信息,可以看到HelloMonica混淆后的名字是a

```
demo.avenwu.net.applymappingdemo.HelloMonica -> demo.avenwu.net.applymappingdemo.a:
    void <init>() -> <init>
    void print() -> a
demo.avenwu.net.applymappingdemo.MainActivity -> demo.avenwu.net.applymappingdemo.MainActivity:
    void <init>() -> <init>
    void onCreate(android.os.Bundle) -> onCreate
```

### apk
![apk-v1]({{ site.baseurl }}/assets/images/v1-package-detail.png)

现在我们把mapping.tx拷贝出来，作为下一次打包用，为了便于管理，重命名为mapping-v1.0.txt;
新增一个HelloWorld.java，并在配置文件proguard-rules.pro中添加applymapping

```
-applymapping mapping-v1.0.txt
```
再次打包，分析得到的mapping和apk，在打包过程中，我们也可以观察日志输出，可以返现，如果配置了applymapping，那么会有相应的日志提示：

```
Obfuscating...
Applying mapping [/Users/aven/work/other/ApplyMappingDemo/app/mapping-v1.0.txt]
Printing mapping to [/Users/aven/work/other/ApplyMappingDemo/app/build/outputs/mapping/release/mapping.txt]...

```

### mapping.txt
```
demo.avenwu.net.applymappingdemo.HelloMonica -> demo.avenwu.net.applymappingdemo.a:
    void <init>() -> <init>
    void print() -> a
demo.avenwu.net.applymappingdemo.HelloWorld -> demo.avenwu.net.applymappingdemo.b:
    void <init>() -> <init>
    void print() -> a
demo.avenwu.net.applymappingdemo.MainActivity -> demo.avenwu.net.applymappingdemo.MainActivity:
    void <init>() -> <init>
    void onCreate(android.os.Bundle) -> onCreate

```

### apk
![apk-v2]({{ site.baseurl }}/assets/images/v2-package-detail.png)

## 小结
从两次打包得到的mapping和apk可以发现，利用applymapping确实可以实现版本之间相同类在混淆是的一致性，新增的文件则则不影响，继续混淆。

