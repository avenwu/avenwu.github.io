---
layout: post
title: "Why did I not use RxJava"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-05-01-01.jpg
keywords: ""
tags: []
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-05-01-01.jpg)

## 前言
从15开始Rx越发火爆，热门到几乎每期的Android Weekly都有它的身影，Android开发者们也乐此不疲谈论着关于Rx的一切；

## RxJava/RxAndroid
Rx的出现使得异步编程编的措手可得，所有的请求，响应，回调处理都可以以一种流式的体验得到实现；可以避免直接使用老的方案，AsyncTask，Handler, ThreadExector，Runnable似乎都变得不直接相关了；

但是正如你所看到的，本文的标题是《Why did I not use RxJava》

为什么我不用Rx开发呢，难道是因为Rx不好用么？  

	Rx太好用了，以至于他让开发者忽略了很多细节，我们只需遵循其提供街开发接口；  
	正是因为他的强封装和透明化，当你一旦开始使用Rx，你的工程就会被同质化，慢慢的同化所有的编码风格形式；

## RxJava是只什么鬼
这是从Github(全球最大的工程师社区:))截取的：

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

在这个物料聚合，流水线处理的过程中就会有一些预定的名词概念，Observable, Operators, Observers, Subscribers, Action，他们代表了在这个生产加工过程中的各个角色。

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

* Observeable

这是整个Rx中最重要的，他是事件产生，中间处理，事件消费整个过程的载体；直译是可被观察的，它实际上就是我们一直在说的“流水线”，流水线都是有输入输出的，Observable提供了不好输入的工厂方法：
	
	just( ) — convert an object or several objects into an Observable that emits that object or those objects
	from( ) — convert an Iterable, a Future, or an Array into an Observable
	repeat( ) — create an Observable that emits a particular item or sequence of items repeatedly
	repeatWhen( ) — create an Observable that emits a particular item or sequence of items repeatedly, depending on the emissions of a second Observable
	create( ) — create an Observable from scratch by means of a function
	defer( ) — do not create the Observable until a Subscriber subscribes; create a fresh Observable on each subscription
	range( ) — create an Observable that emits a range of sequential integers
	interval( ) — create an Observable that emits a sequence of integers spaced by a given time interval
	timer( ) — create an Observable that emits a single item after a given delay
	empty( ) — create an Observable that emits nothing and then completes
	error( ) — create an Observable that emits nothing and then signals an error
	never( ) — create an Observable that emits nothing at all
	
* Operation

进入流水线后，可以开始各种数据处理，转换什么的，这些概括起来就是一堆操作，都有哪些操作呢？  
只能说很多，具体可以直接看Observable的方法摘要，也可以在[Rx官方了解][1]，这里有不同语言实现的各个操作，并且附带示意图.  
常用的有map, flatmap, reduce, all, amb等等

* Observers

Observers是事件的观察者，基本上等同于我们平时说的回调，监听器Listener，在事件通过一系列可选的Operation操作之后，最终这个事件是要被消费掉的，我们通过减价观察者，在Observe回调中得到相应的响应；  
{% highlight java %}
public interface Observer<T> {

    /**
     * Notifies the Observer that the {@link Observable} has finished sending push-based notifications.
     * <p>
     * The {@link Observable} will not call this method if it calls {@link #onError}.
     */
    void onCompleted();

    /**
     * Notifies the Observer that the {@link Observable} has experienced an error condition.
     * <p>
     * If the {@link Observable} calls this method, it will not thereafter call {@link #onNext} or
     * {@link #onCompleted}.
     * 
     * @param e
     *          the exception encountered by the Observable
     */
    void onError(Throwable e);

    /**
     * Provides the Observer with a new item to observe.
     * <p>
     * The {@link Observable} may call this method 0 or more times.
     * <p>
     * The {@code Observable} will not call this method again after it calls either {@link #onCompleted} or
     * {@link #onError}.
     * 
     * @param t
     *          the item emitted by the Observable
     */
    void onNext(T t);

}
{% endhighlight %}
观察者是一个定义好的接口，Observable通过subscribe方法可以添加各种观察者；

* Subscribers

刚讲了观察者，怎么又来一个Subscribers，翻译过来是订阅者，这个怎么看感觉和观察者差不多；  
确实如此，订阅的本身也是观察者，他继承了Observer,但他也包括其他一些接口Subscription，简单说，就是订阅的者可以取消订阅；

### 说好的线程调度呢
现在需要讲一个很重要的东西，那就是前面提到的observeOn和subscribeOn;这两个方法是用来指定Scheduler的，也就是事务执行的具体线程；
RxJava内置了3个默认的调度器：
{% highlight java %}
private final Scheduler computationScheduler;
private final Scheduler ioScheduler;
private final Scheduler newThreadScheduler;
{% endhighlight %}
RxAndroid内置了UI线程的调度器：
{% highlight java %}
/** A {@link Scheduler} which executes actions on the Android UI thread. */
public static Scheduler mainThread() {
    Scheduler scheduler =
            RxAndroidPlugins.getInstance().getSchedulersHook().getMainThreadScheduler();
    return scheduler != null ? scheduler : MainThreadSchedulerHolder.MAIN_THREAD_SCHEDULER;
}
{% endhighlight %}
根据操作的属性，是耗时操作还是UI上的操作，是高IO操作还是高CPU计算操作，我们可以选择不同的调度器，具体可以通过observeOn和subscribeOn指定；

### observeOn vs subscribeOn
上面说到了observeOn和subscribeOn都可以指定操作的调度器，那么这两个方法一样么？不一样的话有什么区别？

要说明这个问题，最好结合一下demo和日志分析，还是hello这个demo，我们在里面补充一下关键的log，看看执行时的线程：
{% highlight java %}
public void hello(String... names) {
    Observable.from(names)
            .map(new Func1<String, String>() {
                @Override
                public String call(String s) {
                    System.out.println(TAG + ":" + s + "-map1-" + Thread.currentThread().toString());
                    return s + "-map1";
                }
            })
            .observeOn(Schedulers.io())
            .map(new Func1<String, String>() {
                @Override
                public String call(String s) {
                    System.out.println(TAG + ":" + s + "-map2-" + Thread.currentThread().toString());
                    return s + "-map2";
                }
            })
            .observeOn(Schedulers.computation())
            .map(new Func1<String, String>() {
                @Override
                public String call(String s) {
                    System.out.println(TAG + ":" + s + "-map3-" + Thread.currentThread().toString());
                    return s + "-map3";
                }
            })
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(new Action1<String>() {

                @Override
                public void call(String s) {
                    System.out.println(TAG + ":Hello " + s + "-" + Thread.currentThread().toString());
                }

            });
}
{% endhighlight %}

{% highlight java %}
MainActivity:Ben-map1-Thread[RxCachedThreadScheduler-1,5,main]
MainActivity:George-map1-Thread[RxCachedThreadScheduler-1,5,main]
MainActivity:Ben-map1-map2-Thread[RxCachedThreadScheduler-2,5,main]
MainActivity:Ben-map1-map2-map3-Thread[RxComputationThreadPool-3,5,main]
MainActivity:George-map1-map2-Thread[RxCachedThreadScheduler-2,5,main]
MainActivity:George-map1-map2-map3-Thread[RxComputationThreadPool-3,5,main]
MainActivity:Hello Ben-map1-map2-map3-Thread[main,5,main]
MainActivity:Hello George-map1-map2-map3-Thread[main,5,main]
{% endhighlight %}

仔细观察日志中的线程名；  
其实subscribeOn和observeOn非常类似，区别主要体现在作用域；

subscribeOn用来指定整个流水线上的操作是在什么线程上执行的，subscribeOn调用的位置可以任意，也就是先于各种Operation都可以；

observeOn发生作用的和subscribeOn不同，他是在调用的时候便生效，可以被调用多次，指定不同的调度器，他的作用域为调用开始之后的所有区间；

当然也可以在命名上加以区分，subscribeOn是订阅，他把流水线上的操作整个订阅到了一个调度器上，observeOn是观察，可以理解为他的优先级更高，如果在某个加工过程比较特殊，他可以零时通过观察的方式把操作挂起到observeOn指定的调度器，相当于被订阅的全局调度器在这里开始失效了，以后也可以多次修改observeOn到不同调度器；

基于他们的性质，在Android开发中，我们一般可以先使用subscribeOn，指定到io等工作调度线程，然后在回调订阅之前observeOn到UI线程的调度器；

## 小结
基本上Rx要理解的东西还是挺多的，本文只是最Rx的基本使用做了介绍，未提及的Operation可以单独在分析；

## 参考
* [https://github.com/ReactiveX/RxJava/wiki](https://github.com/ReactiveX/RxJava/wiki)
* [http://www.grahamlea.com/2014/07/rxjava-threading-examples/](http://www.grahamlea.com/2014/07/rxjava-threading-examples/)

	[1]: http://reactivex.io/documentation/operators.html#alphabetical


