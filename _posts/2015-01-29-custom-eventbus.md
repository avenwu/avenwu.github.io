---
layout: post
title: "EventBus vs Otto vs Guava"
header_image: /assets/img/2016-03-06-30.jpg
description: "分析EventBus，Otto，Guava的实现，并自定义一个简易的Bus"
category: "ioc"
tags: [android]
---
{% include JB/setup %}
![img](/assets/img/2016-03-06-30.jpg)
## 前言
Android有广播和Receiver可以处理消息的传递和响应，要进行**消息-发布-订阅**，除此之外作为开发者现在也有其他类似的方案可以选择，比如EventBus和Otto，都是比较热门的三方库。那么这些三方库到底是怎么实现模块之间的解耦，使得消息可以再不同的系统组件之间传递呢？

## 源码剖析
由于是开源的，完全可以通过分析源代码来了解这些个**消息-发布-订阅**方案在Android内是怎么实现的,下面分别针对EventBus， Otto，Guava简单分析。

### EventBus v2.4.0
从github上获取最新的[EventBus](http://greenrobot.github.io/EventBus/)代码

	
	git clone git@github.com:greenrobot/EventBus.git


直接找到EventBus这个类，从使用角度开始分析：
	
	EventBus的使用可以参考项目wiki，简单的来说就是EventBus#register(),EventBus#post(),实现onEventXXX
	
* EventBus利用反射技术
* register时遍历订阅者，通过反射获取所有public void onEventXXX(XX)方法，构造对应的订阅实例，方便后期post时invoke方法【SubscriberMethodFinder】
* post后通过EventBus#postToSubscription分发至对应线程的事件控制器


### Otto v1.3.6
[Otto](http://square.github.io/otto/)是大名鼎鼎的Square开发的


	git clone git@github.com:square/otto.git


* Otto利用反射和注解
* register时遍历订阅者内的所有方法，根据Subscribe和Produce注解获得所有目标方法，@Subscribe public void xxx(YYY);订阅方法必须是public且只有一个参数
* post后在EventHandler#handleEvent内invoke事件，这里没有EventBus中的线程区分，默认是MainThread，也可以任何线程ANY，但是必须是一致的ThreadEnforcer

从代码上来看EventBus和Otto非常像，不知道EventBus作者在设计编码时是否参考了Otto得设计，Otto项目则明确表示其是基于Google的Guava而来，Guava是Google开发的一个工具类库包含了非常多的实用工具类，其中就有一个EventBus模块，但是这个EventBus是并没有针对Android平台做线程方面的考量。

所以三者的是有关联的：

* Guava EventBus首先实现了一个基于发布订阅的消息类库，默认以注解来查找订阅者
* Otto借鉴Guava EventBus，针对Android平台做了修改，默认以注解来查找订阅、生产者
* EventBus和前两个都很像，v2.4后基于反射的命名约定查找订阅者，根据其自己的说法，效率上优于Otto，当然我们测试过，这也不是本文的重点。

由于源代码也不少，所以只列举了核心代码对应的位置，感兴趣的童鞋肯定会自己去研读。

## 自定义一个EventBus

上面的库当然不是为了研究而研究，现在理解了他们的核心思路后，我们其实已经可以着手自己写一个简单版的**消息-发布-订阅**。  
现在先定义要实现的程度：


* 基于UI线程的**消息-发布-订阅**
* 使用上合EventBus尽量保持一致，比如register，post，onEvent

### 思路设计

一个简单的Bus大致上需要有几个东西，Bus消息中心，负责绑定/解绑，发布/订阅;Finder查找定义好的消息处理方法；PostHandler分发消息并处理.

![bus_structure](/assets/bus_structure.png)

### Bus实现

这里的Bus做成单例，这样无论在什么地方注册，发布都是有这个消息中心来处理。  
用一个Map来保存我们的订阅关系，当消息到达时从map中取出该消息类型的所有订阅方法，通过反射依次invoke。

{% highlight java %}
public class Bus {

    static volatile Bus sInstance;

    Finder mFinder;

    Map<Class<?>, CopyOnWriteArrayList<Subscriber>> mSubscriberMap;

    PostHandler mPostHandler;

    private Bus() {
        mFinder = new NameBasedFinder();
        mSubscriberMap = new HashMap<>();
        mPostHandler = new PostHandler(Looper.getMainLooper(), this);
    }

    public static Bus getDefault() {
        if (sInstance == null) {
            synchronized (Bus.class) {
                if (sInstance == null) {
                    sInstance = new Bus();
                }
            }
        }
        return sInstance;
    }

    public void register(Object subscriber) {
        List<Method> methods = mFinder.findSubscriber(subscriber.getClass());
        if (methods == null || methods.size() < 1) {
            return;
        }
        CopyOnWriteArrayList<Subscriber> subscribers = mSubscriberMap.get(subscriber.getClass());
        if (subscribers == null) {
            subscribers = new CopyOnWriteArrayList<>();
            mSubscriberMap.put(methods.get(0).getParameterTypes()[0], subscribers);
        }
        for (Method method : methods) {
            Subscriber newSubscriber = new Subscriber(subscriber, method);
            subscribers.add(newSubscriber);
        }
    }

    public void unregister(Object subscriber) {
        CopyOnWriteArrayList<Subscriber> subscribers = mSubscriberMap.remove(subscriber.getClass());
        if (subscribers != null) {
            for (Subscriber s : subscribers) {
                s.mMethod = null;
                s.mSubscriber = null;
            }
        }
    }

    public void post(Object event) {
        //TODO post with handler
        mPostHandler.enqueue(event);
    }
}
{% endhighlight %}

### Finder
查找订阅方法即可以用注解，也可以用命名约定，这里先实现命名约定的方式。  
为了处理方便这里和EventBus不完全一致，只做了方法名和参数的限制，但是最好实现的严谨些。

{% highlight java %}
public class NameBasedFinder implements Finder {

    @Override
    public List<Method> findSubscriber(Class<?> subscriber) {
        List<Method> methods = new ArrayList<>();
        for (Method method : subscriber.getDeclaredMethods()) {
            if (method.getName().startsWith("onEvent") && method.getParameterTypes().length == 1) {
                methods.add(method);
                Log.d("findSubscriber", "add method:" + method.getName());
            }
        }
        return methods;
    }
}
{% endhighlight %}

### PostHandler
分发消息肯定要用到Handler，EventBus中自己维护了一个队列来来处理消息的入栈、出栈，我这里就世界用了Message来传递

{% highlight java %}
public class PostHandler extends Handler {

    final Bus mBus;

    public PostHandler(Looper looper, Bus bus) {
        super(looper);
        mBus = bus;
    }

    @Override
    public void handleMessage(Message msg) {
        CopyOnWriteArrayList<Subscriber> subscribers = mBus.mSubscriberMap.get(msg.obj.getClass());
        for (Subscriber subscriber : subscribers) {
            subscriber.mMethod.setAccessible(true);
            try {
                subscriber.mMethod.invoke(subscriber.mSubscriber, msg.obj);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    void enqueue(Object event) {
        Message message = obtainMessage();
        message.obj = event;
        sendMessage(message);
    }
}
{% endhighlight %}

## 小结

基本上的代码都在这里，实现一个Bus还是挺简单的，当然如果吧各种情况都考虑进去就会变得复杂一些，比如支持多线程线程，也不可能想本文这样区区数百行代码就搞定。
感兴趣的可以到这里获取上面自定义bus的源代码：[https://github.com/avenwu/support/tree/master/support/src/main/java/net/avenwu/support](https://github.com/avenwu/support/tree/master/support/src/main/java/net/avenwu/support/eventbus)
