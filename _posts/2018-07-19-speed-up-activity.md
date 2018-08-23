---
layout: post
title: "页面速度分析与优化实操"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2018-08-03.jpg
keywords: ""
tags: []
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2018-08-03.jpg)

## 前言

为了更好的用户体验，在App运行过程中我们往往对页面加载的速度和卡顿都是比较敏感的，本文基于58同城首页启动优化调研，分享一下如何诊断特定页面并提高加载速度。

## 问题背景

凡是都一个契机，先讲一下问题背景：

> 通过外部吊起和开屏广告点击，我们可以直达58App的一个具体落地页，展示详情页面。为了用户体验，我们采取的是直达落地页，这种情况下应用的主页界面或者首页是不存在的。当我们退出落地页后，希望用户回到主界面。这个时候由于我们主界面比较重，整个返回和展示首页的时间比较长，直观感受在1到2秒左右。

为了解决这个问题，有两种思路：

1. 对首页做预加载，比如出落地页之前或之后，我们把首页也打开；
2. 优化首页自身的启动速度，使其满足我们的体验要求；

方案一其实也比较常见，预加载可以使资源得到提前处理。他比较适合加载过程本身时间不可优化的情形。在这里，如果我们提前加载首页，有可能会出现在落地页的时候整个操作更加卡顿，特别是在低端机型上，本身前台要加载一个页面，同时还要预加载另一个页面，这里面的耗时与时机把握都是比较棘手的。

方案二，如果首页自身的加载，如果能优化到满足体验要求，那么问题就直接解决了。

## CPU&线程诊断

首先我们需要量化一个页面加载过程中的时间消耗情况。最简单的可以使用Android SDK提供的systrace，工具的使用大家可以参考[开发者文档](https://developer.android.com/studio/command-line/systrace)。

![home](http://7u2jir.com1.z0.glb.clouddn.com/img/systrace-home.png)

可以看到问题比较多，有线程竞争CPU导致主线程饥饿，也有View绘制过重问题，还有Bitmap加载问题等；

针对线程饥饿问题，实际上就是主线程被阻塞了，解决这类问题，可以通过查看问题发生的时间偏上，占用CPU的线程，并找到对应线程降低它的优先级。

```
Alert
Scheduling delay
Running	
568.782 ms
Not scheduled, but runnable	
126.276 ms
Sleeping	
44.083 ms
Blocking I/O delay	
16.049 ms
Frame	
Description	
Work to produce this frame was descheduled for several milliseconds, contributing to jank. Ensure that code on the UI thread doesn't block on work being done on other threads, and that background threads (doing e.g. network or bitmap loading) are running at android.os.Process#THREAD_PRIORITY_BACKGROUND or lower so they are less likely to interrupt the UI thread. These background threads should show up with a priority number of 130 or higher in the scheduling section under the Kernel process.
```

![main-thread-wait](http://7u2jir.com1.z0.glb.clouddn.com/img/main-thread-wait.png)

在这上图中，我们发现一段时间片内有各种线程挤占了全部4个CPU，导致主线程出现一段饥饿等待的情况。根据前面的说明信息和时间图表，主线程的优先级是116，默认service的优先级大概率是120，普通线程的优先级也是120，那些低于116的基本都是系统级别的线程。我们能做的主要是降低非系统线程的优先级。

通过查看红圈1，2，3的线程优先级, 需要把其中priority小于130的线程提高到大于等于130。

这里面的难点在于如果根据图中的name来找到代码中对应的线程。有些线程形如Thread-xxx的是默认线程名字，也是比较难定位的。这时候急需要借助其他工具，比如Trace和Profiler来锁定代码段。

## 锁定代码

为了更进一步的发现这些线程对应的代码逻辑，我们可以借助Android Studio的Profiler分析，CPU和内存。

![profiler-thread](http://7u2jir.com1.z0.glb.clouddn.com/img/profiler-thread.png)

确定了具体占用CPU的线程后，可以对相应的线程调整优先级；

## 剥离耗时任务

除了异步线程竞争，还需要减轻主线程的负担，通过观察UI Thread上的事件，可以知道那一段比较耗时，如果需要检测指定方法，我们可以手工添加Trace:

```java
Trace.beginSection("XX#onCreateView");
// TODO
Trace.endSection();
```

![trace-section](http://7u2jir.com1.z0.glb.clouddn.com/img/trace-section.png)

这里主要就是根据发现的耗时代码段，进行剥离了，比如一个方法内联系N次有IO读写，就是一个疑点，把这些累积耗时的操作批量迁移到工作线程。

## 延迟加载

为了使界面加载更快，可以将整个页面进行拆分，优先展示主框架，比如顶部栏，tab栏等，内容区可以延后展示。

![home](http://7u2jir.com1.z0.glb.clouddn.com/img/device-2018-08-03-155810.gif)

## 小结

优化是无止境的，通过各种技术手段达到优化预期只是一小步。
