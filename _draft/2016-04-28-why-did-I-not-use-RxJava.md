---
layout: post
title: "Why did I not use RxJava"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-04-18-02.jpg
keywords: ""
tags: []
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-04-18-02.jpg)

## 前言
从15开始Rx越发火爆，热门到几乎每期的Android Weekly都有它的身影，Android开发者们也乐此不疲谈论着关于Rx的一切；

## RxJava/RxAndroid
Rx的出现使得异步编程编的措手可得，所有的请求，响应，回调处理都可以以一种流式的体验得到实现；可以避免直接使用老的方案，AsyncTask，Handler, ThreadExector，Runnable似乎都变得不直接相关了；

但是正如你所看到的，本文的标题是《Why did I not use RxJava》

为什么我不用Rx开发呢，难道是因为Rx不好用么？  

	Rx太好用了，以至于他让开发者忽略了很多细节，我们只需遵循其提供街开发接口；  
	正是因为他的强封装和透明化，当你一旦开始使用Rx，你的工程就会被同质化，慢慢的通化所有的编码风格形式；

## RxJava是只什么鬼
这是从全球最大的工程师社区截取的：

```
RxJava is a Java VM implementation of Reactive Extensions: a library for composing asynchronous and event-based programs by using observable sequences.

It extends the observer pattern to support sequences of data/events and adds operators that allow you to compose sequences together declaratively while abstracting away concerns about things like low-level threading, synchronization, thread-safety and concurrent data structures.
```

简言之RxJava是为异步和事件驱动而生的，利用的是经典的观察者模式，同时是基于Java VM实现的（这个没有深扒过，不知道细节）

## RxJava上手
在很久之前，写过一篇Rx相关的文章[RxJava/Retrolambda with Android]({{ site.baseurl }}/2015/06/25/rxjavaretrolambda-with-android)，不过没有过多涉及Rx的使用；


# 参考
* [https://github.com/ReactiveX/RxJava/wiki](https://github.com/ReactiveX/RxJava/wiki)



