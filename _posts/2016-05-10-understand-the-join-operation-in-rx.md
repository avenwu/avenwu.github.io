---
layout: post
title: "如何理解RxJava中的join操作"
description: ""
header_image: /assets/img/2016-05-01-01.jpg
keywords: ""
tags: [RxJava]
---
{% include JB/setup %}
![img](/assets/img/2016-05-01-01.jpg)

## 前言
先前写过一篇文章，介绍Rx中不容易理解的概念([Rx那些不怎么好理解的点]({{ site.baseurl}}/2016/05/05/confusing-concept-in-rx/))，但是并没有涵盖全，本文一起来看一下join的含义(如果你已经很清楚Join的含义那么可以跳过本文了)

## 合并多个Observable
Rx有很多预定义的针对Observable的Operation，根据操作的类型可以分为过滤，变换，组合，而join属于组合这一类；  
组合实际上就是有多个Observable，但是我们希望的是多输入单输出，也就是多对一；那么根据合并的不同特性有merge，zip，join等等；

* merge按顺序合并两个Observable发出的数据
* zip按顺序打包两个Obserable发出数据的合体；
那么join呢？  
在阅读了RxJava Essential和Reactivex.io上的解释后，还是很难理解,官方的示例图也是基本没看明白：
![join](https://raw.github.com/wiki/ReactiveX/RxJava/images/rx-operators/join_.png)

总算皇天不负有心人，在反复查阅英文资料后终于在Rx的起源处C# Rx中找到了能理解的说明[http://www.introtorx.com/uat/content/v1.0.10621.0/17_SequencesOfCoincidence.html#Join](http://www.introtorx.com/uat/content/v1.0.10621.0/17_SequencesOfCoincidence.html#Join)

{% highlight shell %}
When left produces a value, a window is opened. That value is also then passed to the leftDurationSelector function. The result of this function is an IObservable<TLeftDuration>. When that sequence produces a value or completes then the window for that value is closed. Note that it is irrelevant what the type of TLeftDuration is. This initially left me with the feeling that IObservable<TLeftDuration> was all a bit over kill as you effectively just need some sort of event to say 'Closed'. However by allowing you to use IObservable<T> you can do some clever stuff as we will see later.

So let us first imagine a scenario where we have the left sequence producing values twice as fast as the right sequence. Imagine that we also never close the windows. We could do this by always returning Observable.Never<Unit>() from the leftDurationSelector function. This would result in the following pairs being produced.

Left Sequence

L 0-1-2-3-4-5-
Right Sequence

R --A---B---C-
0, A
1, A
0, B
1, B
2, B
3, B
0, C
1, C
2, C
3, C
4, C
5, C
As you can see the left values are cached and replayed each time the right produces a value.
{% endhighlight %}

说白了join操作我们不能把两个Observable等价看待，join的效果类似于排列组合，把第一个数据源A作为基座窗口，他根据自己的节奏不断发射数据元素，第二个数据源B，每发射一个数据，我们都把它和第一个数据源A中已经发射的数据进行一对一匹配；

举例来说，如果某一时刻B发射了一个数据“B”,此时A已经发射了0，1，2，3共四个数据，那么我们的合并操作就会把“B”依次与0,1,2,3配对，得到四组数据：
0, B
1, B
2, B
3, B

是不是和简单？可惜我在在花费大量时间研究出join本质前，一直无法理解，希望本文能帮助到遇到同样问题的人；

## 小结
所以join得本质就是这么简单，但是很遗憾在RxJava的各种介绍和Github上面对join都是一笔带过，并没有把他的概念讲清楚，如果读者在细致的学习RxJava时就会遇到很多问题；

