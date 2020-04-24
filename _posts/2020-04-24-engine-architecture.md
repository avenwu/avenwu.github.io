---
layout: post
title: "笔记|Flutter引擎架构"
description: ""
header_image: /assets/img/2020-04-24-01.jpeg
keywords: ""
tags: [Flutter]
---
{% include JB/setup %}
![img](/assets/img/2020-04-24-01.jpeg)

* 目录
{:toc #markdown-toc}

## 背景

在Flutter仓库中，有一些官方资料，比如Flutter引擎里的线程模型就有一篇官方资料。原文地址如下：

https://github.com/flutter/flutter/wiki/The-Engine-architecture#threading


## 概览

> Q1: Flutter Engine到底是什么

一款高质量，可移植的移动设备运行时。引擎实现了Flutter的核心库，包括动画，图形，文件，网络IO，可访问性（accessibility）支持，插件架构；以及一个Dart的运行时，开发，编译，运行Flutter程序需要的工具链。

整个引擎使用的一些核心技术：

* Skia：2D图形渲染库，
* Dart：支持垃圾回收的，面向对象的虚拟机

这些东西都涵盖在一个Shell壳子里面。不同平台有不同的壳，比如Android和iOS有各自的Shell层实现。

除此之外还提供了一套emberdder API，允许引擎被当做一个库来使用。

> Q2: Shell壳都干了些什么事情

Shell是平台相关的实现层，比如输入法IMEs的交互，app生命周期事件。在Shell里面，提供了dart:ui库来支持对SKia底层能力的调用。同时可以绕过引擎，使用Platform Channel与Dart侧通信。

Flutter可以看成是三层架构

* Framework 由Dart实现
* Engine 由C/C++实现
* Embedder 平台相关

![](https://raw.githubusercontent.com/flutter/engine/master/docs/flutter_overview.svg?sanitize=true)

## 线程Threading

Flutter引擎本身并不创建也不管理自己的线程，线程的管控是由embedder层负责的，包括他们的消息循环。Embedder对engnie提供了执行任务的runner需要的线程。处理Embedder的线程，Dart虚拟机也有自己的一套线程池，这块线程池中的线程对Engine和Embedder都是不可访问的。

引擎要求Embedder提供4个task runner的引用，但是并不关心每个runner所在的线程是否是独立的。处于更优的性能，每个runner分配独立线程更好。

* Platform Task Runner =》平台的主线程，如Android的main线程
* UI Task Runner
* GPU Task Runner
* IO Task Runner

> 每个引擎有一份独立的UI, GPU， IO Runnter独立线程；所有共享引擎共用platform runner的线程

### Platform Task Runner

虽然Platform Task Runner是跑在宿主的主线程上面，但是引擎对这个线程并没有特别含义，实际上可以在不同线程上面启动多个引擎。

和引擎的交互动作都必须在Platform线程上执行。经典的MethodChannel回调onMethodCall即在主线程上面。因此不能执行耗时操作阻塞主线程



### UI Task Runner

引擎执行root Isolate中Dart代码的线程。和App主线程不同。root Isolate是一个特殊的Isolate，他有一些与Flutter相关的binding函数。这些绑定关系被Engine配置好后，用于调度和提交frame帧，每一帧Flutter需要绘制：

* root Isolate告知引擎需要绘制一帧了
* 引擎想平台发起通知，下一次vsync需要更新
* 平台等待下一次vsync
* 当vsync消息触发时，引擎唤醒Dart代码
  * 更新动画差值器
  * 在build阶段rebuild widget
  * 将新构建的widget布局并绘制layer树，此时没有真正栅格化操作，只是一些描述需要在pait阶段绘制的信息
  * 构建或更新节点树，其中包含使用性的信息

除了构建引擎需要绘制的帧指纹，root Isolate也执行对平台插件消息，计时器，微任务和异步I / O（来自套接字，文件句柄等）的所有响应。

总结一下，由于UI Runner是屏幕上绘制内容源的构建者，对UI Runner的同步耗时阻塞会导致应用的卡顿，即便是几毫秒也足以丢失一帧。长时间操作一般来说只能是由于Dart代码本身，因为引擎不会将native代码任务放到这个runner上，所以这个runner也可以被称为Dart 线程。虽然Embedder有能力将其他任务丢到这个runner上，单不建议这么做，因为会导致页面卡顿。

如果在Dart中执行耗时任务是无法避免的，那么建议使用一个独立的Isolate来处理，例如通过compute方法。在独立Isolate中执行的代码，实际上会运行在由Dart虚拟机管理的线程池中。

终止root isolate会同时终止由root isolate孵化出阿里的的所有isolate。另外需要注意的是，非root isolate确实基本的帧调度绑定关系，所以将不能与Flutter框架交互。

### GPU Task Runner

执行需要访问设备GPU的任务，跟UI Task Runner创建的layer tree，去生成适当的GPU指令。GPU Runner同时也负责为准备每一帧所需的所有GPU资源，包括与platform通信以准备好framebuffer，管理Surface生命周期，确保帧所需的纹理和缓冲区都完全准备好了。

类似的GPU Runner也不能耗时，如果处理layer tree，展示帧耗时过长，会影响下一帧在UI runner上的调度。

UI 和GPU Runner的耗时都会导致页面的卡顿Jank

### IO Task Runner

由于其他三个Runner都是时间敏感的，但是有些耗时任务是GPU需要的，这时候就可以利用IO Runner处理。

IORunner的核心功能是读取压缩图片，比如冲asset中读取，保证这些图片准备好给GPU Runner渲染。为了保证纹理已经可用，第一步需要将压缩数据（通常是PNG,JPEG）读取为blob数据块，然后解压为对GPU友好的数据格式，然后上传到GPU上。这些动作都是耗时操作，因此不能在GPUrunner上处理。由于只有GPU Runner才可以操作GPU，所以IO Runner在引擎配置阶段就会设置一个特殊的上下文，他与GPU Runner处于同一个共享组内。

在实现中，读取压缩字节和解压动作可以发生在线程池中。IO Runner只有在特定线程下访问上下文才是安全的。ui.Image获取资源的唯一办法技术异步调用

IO Runner所在线程也无法被Dart或Native插件直接访问。

## 小结
标题写的是引擎架构，从原文翻译而来，但实际上资料中介绍的主要是还只是引擎的线程模型和对应起的作用。

