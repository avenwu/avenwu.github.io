---
layout: post
title: "Rx里哪些不怎么好理解的Operation"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-04-18-02.jpg
keywords: ""
tags: []
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-04-18-02.jpg)

## 前言
今天得空，在下班前看了看之前遗留下/不理解的Rx概念；

## Debounce or Sample
学习Rx的相关知识，一定要去[Reactive.io](http://reactivex.io/documentation/operators/debounce.html), 但是我在看了debounce于sample的说明后一头雾水:
debounce的原因大致是“去抖动; 防抖动; 弹跳;”，sample的意思是采样，抽样；从Reactivex.io上的介绍来看，两者非常相似；
有一个显著差别是，
	
1. debounce函数过滤掉由Observable发射的速率过快的数据；如果在一个指定的时间间隔过去了仍旧没有发射一个，那么它将发射最后的那个。
[debounce](http://reactivex.io/documentation/operators/images/debounce.png)

2. sample定期从Observable发射的数据中去最后一个，如果某个时期内没有数据发射，那么这段时期内就没有被观察的数据，他不会发射最后一个；
[sample](http://reactivex.io/documentation/operators/images/sample.png)


## 


