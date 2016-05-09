---
layout: post
title: "Rx那些不怎么好理解的点"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-05-01-01.jpg
keywords: ""
tags: []
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-05-01-01.jpg)

## 前言
学习Rx的相关知识，一定要去[Reactive.io](http://reactivex.io/documentation/operators/debounce.html), 但是我在看了其中的一些operation的说明后，有几个操作让人一头雾水:

## Debounce or Sample
debounce的原因大致是“去抖动; 防抖动; 弹跳;”，sample的意思是采样，抽样；从Reactivex.io上的介绍来看，两者非常相似；
有一个显著差别是，
	
1. debounce函数过滤掉由Observable发射的速率过快的数据；如果在一个指定的时间间隔过去了仍旧没有发射一个，那么它将发射最后的那个。
![debounce](http://reactivex.io/documentation/operators/images/debounce.png)

2. sample定期从Observable发射的数据中取最后一个，如果某个时期内没有数据发射，那么这段时期内就没有被观察的数据，他不会发射最后一个；
![sample](http://reactivex.io/documentation/operators/images/sample.png)


## First or Single
两者都是只从数据源中返回一个符合条件的数据，但是single在Observable结束时都没有合适的数据时会抛出一个NoSuchElementException异常
![first](http://reactivex.io/documentation/operators/images/first.png)

![single](http://reactivex.io/documentation/operators/images/single.png)


## Event Bus
在Rx流行起来之前很多开发都接触过EventBus(事件总线),事件总线用的是发布-订阅模式，和Rx的观察者模式非常相像，不过事件总线更加关注消息的广播，在解耦这件事上大家都是通过中间人在处理，从而达到消息生成和消费之间无关。  
在引入Rx之后我们甚至也能实现RxBug，现在我们一起看一下：
{% highlight java %}
public class RxBus {

  private final Subject<Object, Object> _bus = new SerializedSubject<>(PublishSubject.create());

  public void send(Object o) {
    _bus.onNext(o);
  }

  public Observable<Object> toObserverable() {
    return _bus;
  }
}
@OnClick(R.id.btn_demo_rxbus_tap)
public void onTapButtonClicked() {

    _rxBus.send(new TapEvent());
}
_rxBus.toObserverable()
    .subscribe(new Action1<Object>() {
      @Override
      public void call(Object event) {

        if(event instanceof TapEvent) {
          _showTapText();

        }else if(event instanceof SomeOtherEvent) {
          _doSomethingElse();
        }
      }
    });
{% endhighlight %}
短短几行代码就可以实现事件总线中的中间人角色；

![PublishSubject](https://raw.github.com/wiki/ReactiveX/RxJava/images/rx-operators/S.PublishSubject.png)

## 参考
* [http://reactivex.io/documentation/operators/debounce.html](http://reactivex.io/documentation/operators/debounce.html)
* [http://nerds.weddingpartyapp.com/tech/2014/12/24/implementing-an-event-bus-with-rxjava-rxbus/](http://nerds.weddingpartyapp.com/tech/2014/12/24/implementing-an-event-bus-with-rxjava-rxbus/)
