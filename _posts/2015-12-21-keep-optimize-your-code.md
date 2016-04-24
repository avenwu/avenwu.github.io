---
layout: post
title: "为什么要优化你的代码？"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-08.jpg
keywords: "优化 代码"
description: "为什么要优化你的代码？很多时候其实你是最后一道防线，如果你不做，就没有人做"
category: 
tags: [优化]
---
{% include JB/setup %}

![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-08.jpg)

## 前言

	话说“夜路走多了，总会碰到鬼”

有一段时间没做笔记了，本文聊一聊工程开发中的后续工作；

笔者从事移动开发工作第四个年头，前前后后也接触了不少项目，在项目开发迭代的大背景下，“代码腐化“问题随着时间推移会最终显现出来；
这里不说谈为什么会出现”腐化“问题，因为原因真的很多，而且往往不可预知；


## 做与不做

	面对上述问题，持续优化是一种吃力但是有效的解决方案，可惜没有多少团队真正实施下来；
	从项目本身而言，其目标是用户体验，工程的优化是技术内在，往往不具备立竿见影的体验；
	开发的工程师如果没有绝对的动力，谁愿意贸然改动本就”正常“运行的代码，改坏了怎么整？


所以所说没点闲工夫和勇气，动刀子的事是不容易的；

	但是很多时候其实你是最后一道防线，如果你不做，就没有人做；

## 该出手时就出手
笔者3月份入职极客学院，打一波广告，不喜勿喷:
	
	极客学院IT在线教育平台-中国最大的IT职业在线教育平台

公司Leader也开明，因此Android端已经被我"折腾"好一阵, 这也是我继续在公司效力的动力之一”有舞台施展“；
经历了多轮调整和优化，目前项目的架构和开发总体朝着更完善的方向前进；

极客学院的android客户端近期即将上线的4.0版本，由笔者主导开发的版本数也++;

笔者优化代码一般在是版本迭代之间进行，一来不收时间约束，效果好就集成至仓库，不好的话就直接废弃；而来优化这种活可能会影响业务，所以在闲时做的优化可以在下一版测试当中被充分考验；

在新版上线的空隙间正好又是一次练兵优化的时机，此次目标是优化接口请求的业务成调用；

声名在外的”Retrofit“之所以广受好评，其简洁的调用，使得快速开发成为可能功不可没；极客学院内部的就业版客户端用的正是这套api框架；

主站极客学院app使用的时Apache的http，功能满足需求，因此一直用到现在，此次的优化主要是为了封装调用层，让业务逻辑也想Retrofit一样可以通过非常简单的注解就实现接口定义；

先来看看根据需求定制的几个注解类，包含了get，post和cache等：

![http://7u2jir.com1.z0.glb.clouddn.com/category_annotations.jpg](http://7u2jir.com1.z0.glb.clouddn.com/category_annotations.jpg)

接口定义的时候，也很简单：

![http://7u2jir.com1.z0.glb.clouddn.com/api_demo.jpg](http://7u2jir.com1.z0.glb.clouddn.com/api_demo.jpg)

接口调用,包括单元测试和业务调用:

![http://7u2jir.com1.z0.glb.clouddn.com/copyright_test.jpg](http://7u2jir.com1.z0.glb.clouddn.com/copyright_test.jpg)

![http://7u2jir.com1.z0.glb.clouddn.com/copyright_api.jpg](http://7u2jir.com1.z0.glb.clouddn.com/copyright_api.jpg)

基本上写一个api请求就是这样简单几步就搞定，和retrofit用起来差不多；

## 结束语

自己写的代码，含着泪也要优化下去😄

---
