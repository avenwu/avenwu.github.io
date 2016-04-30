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

### RxJava上手
在很久之前，写过一篇Rx相关的文章[RxJava/Retrolambda with Android]({{ site.baseurl }}/2015/06/25/rxjavaretrolambda-with-android)，不过没有过多涉及Rx的使用；

Github上的示例是这么样的：  
{% highlight java %}
public static void hello(String... names) {
    Observable.from(names).subscribe(new Action1<String>() {

        @Override
        public void call(String s) {
            System.out.println("Hello " + s + "!");
        }

    });
}

{% endhighlight %}

{% highlight java %}
hello("Ben", "George");
Hello Ben!
Hello George!
{% endhighlight %}

### RxJava设计/使用思想
以上示例，无非是是一个数组内容串行了，然后可以对每一个元素进行操作，其实这就是Rx核心思想体现，这有一张示意图：  
![from.png](https://raw.github.com/wiki/ReactiveX/RxJava/images/rx-operators/from.png)

	所以Rx使用设计思路实际上类似于生产流水线，把需要的原料通过Observable的工厂方法有序的分发到流水线上，方便后续加工；

在这个物料聚合，流水线处理的过程中就会有一些预定的名词概念，Observable, operators, Observers, Subscribers, Action，他们代表了在这个生产加工过程中的各个角色。

Rx的来料加工可以很便捷得指定加工操作的线程问题(Schelduler)；同时可以任意的对来料进行修改，包装，转换，即输入时方形，可以先map一下转成圆形，再交给下一道流程（operation）；  
![schedulers](http://reactivex.io/documentation/operators/images/schedulers.png)

所以Rx的特长是线程调度和流水线操作，针对单一的对象的单一操作来说到时没什么使用的必要的优势，所以前面给出的Demo实际上并不能明确的让开发者体验到Rx的好处,反而容易觉得Rx造成代码冗余;

### 感受Rx的优势
现在我们把实例改造一下，以便更直观的看出Rx能做的事情；  
{% highlight java %}
public void hello(String... names) {
    Observable.from(names)
            .subscribeOn(Schedulers.io())
            .map(new Func1<String, String>() {
                @Override
                public String call(String s) {
                    return s + "-map1";
                }
            })
            .map(new Func1<String, String>() {
                @Override
                public String call(String s) {
                    return s + "-map2";
                }
            })
            .map(new Func1<String, String>() {
                @Override
                public String call(String s) {
                    return s + "-map3";
                }
            })
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(new Action1<String>() {

                @Override
                public void call(String s) {
                    System.out.println(TAG + ":Hello " + s + "-" + Thread.currentThread().toString());
                }

            });
}
{% endhighlight %}
上述的操作，输入任然是两个name，但是我们故意对name做了三次修改，简单起见每次都通过map,为name增补一个后缀；
{% highlight java %}
hello("Ben", "George");
MainActivity:Hello Ben-map1-map2-map3-Thread[main,5,main]
MainActivity:Hello George-map1-map2-map3-Thread[main,5,main]
{% endhighlight %}

这里对name的修改只是模拟对输入数据的修改，实际中可能会更复杂，比如需要更加输入的name查询数据可得到他的详细个人数据，最后在打印出来；
Rx在处理事件流是会显得特别得心应手，细心的话可以发现，相比较官方的demo，这里的增强版多了一个subscribeOn和observeOn；

	那么这两个方法是做什么用的，有什么区别么？
### Rx名称概念一箩筐
现在看一下Rx涉及到的几个很重要的名词。  

## 参考
* [https://github.com/ReactiveX/RxJava/wiki](https://github.com/ReactiveX/RxJava/wiki)
* [http://www.grahamlea.com/2014/07/rxjava-threading-examples/](http://www.grahamlea.com/2014/07/rxjava-threading-examples/)


