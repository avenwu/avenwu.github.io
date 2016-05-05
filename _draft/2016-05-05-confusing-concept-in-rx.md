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

## Debounce
学习Rx的相关知识，一定要去[Reactive.io](http://reactivex.io/documentation/operators/debounce.html), 但是我在看了debounce的说明后一头雾水:

	only emit an item from an Observable if a particular timespan has passed without it emitting another item

查了下debounce的原因大致是“去抖动; 防抖动; 弹跳;”，但是我怎么也断不好上面的原文说明语句；  

今天忽然开窍了，其实debounce是过滤，过滤约定时间之内的所有发射的数据，比如说流水生产线上会不定时的向下游发数据，但是debounce会在下游放一个拦截的挡板，每隔一分钟挡板会放开一次，在一分钟没到前到达的过量的数据都会被拦截住，等一分钟到了，挡板会放行最后到达的一个数据，那么其他的数据就相当于后浪吧前浪拍死在沙滩上了；


