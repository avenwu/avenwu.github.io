---
layout: post
title: "OkHttp【四】任务调度Dispatcher"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2017-12-26-01.png
keywords: ""
tags: [okhttp]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2017-12-26-01.png)

## 前言

在发起HTTP请求后，`OkHttp`在`RealCall`封装的相关逻辑内执行了请求发起动作，而负责记录和调度`Call`的则是`Dispatcher`。
本文一起分析`OkHttpClient#Dispatcher`的相关实现。

```java
/**
 * Policy on when async requests are executed.
 *
 * <p>Each dispatcher uses an {@link ExecutorService} to run calls internally. If you supply your
 * own executor, it should be able to run {@linkplain #getMaxRequests the configured maximum} number
 * of calls concurrently.
 */
 public final class Dispatcher {
     // ...
 }
```

## 任务调度 

在构造`OkHttpClient`实例的时候，通过构造函数，创建了一个请求调度类`Dispatcher`。该类会在`RealCall`的异步请求接口`enqueue`和同步请求接口`execute`中被执行

```java
client.dispatcher().executed(this);
```

```java
client.dispatcher().enqueue(new AsyncCall(responseCallback));
```

先看下`Dispatcher`的内部主要接口和相关关系

![OkHttp-Dispatcher.png](http://7u2jir.com1.z0.glb.clouddn.com/img/OkHttp-Dispatcher.png)

可以看到，`Dispatcher`主要用于处理异步请求，同步请求只是简单加入了`Deque`。

## 并发限制

执行网络请求用的是Java并发包提供的API, 这里实例化一个`ThreadPoolExecutor`来处理多线程任务：

```java
public synchronized ExecutorService executorService() {
    if (executorService == null) {
    executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
        new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
}
```

从参数的选用上来看，上面的线程池声明也可以用并发包中`newCachedThreadPool`方法，配置上自身的`ThreadFactory`：

```java
/**
* Creates a thread pool that creates new threads as needed, but
* will reuse previously constructed threads when they are
* available, and uses the provided
* ThreadFactory to create new threads when needed.
* @param threadFactory the factory to use when creating new threads
* @return the newly created thread pool
* @throws NullPointerException if threadFactory is null
*/
public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                    60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>(),
                                    threadFactory);
}
```

根据`ThreadPoolExecutor`的定义，我们知道这里实例化了一个没有限制的无限队列来承载请求任务，按需创建/复用线程。无限制队列的特点就是，理论上提交的任务不断积累时，最终将耗尽内存。因此相对来说我们其实更常用的是有限队列，通过舍弃一些任务或者拒绝新增任务来保证机器不会耗尽内存。

```java
/**
* Creates a new {@code ThreadPoolExecutor} with the given initial
* parameters and default rejected execution handler.
*
* @param corePoolSize the number of threads to keep in the pool, even
*        if they are idle, unless {@code allowCoreThreadTimeOut} is set
* @param maximumPoolSize the maximum number of threads to allow in the
*        pool
* @param keepAliveTime when the number of threads is greater than
*        the core, this is the maximum time that excess idle threads
*        will wait for new tasks before terminating.
* @param unit the time unit for the {@code keepAliveTime} argument
* @param workQueue the queue to use for holding tasks before they are
*        executed.  This queue will hold only the {@code Runnable}
*        tasks submitted by the {@code execute} method.
* @param threadFactory the factory to use when the executor
*        creates a new thread
* @throws IllegalArgumentException if one of the following holds:<br>
*         {@code corePoolSize < 0}<br>
*         {@code keepAliveTime < 0}<br>
*         {@code maximumPoolSize <= 0}<br>
*         {@code maximumPoolSize < corePoolSize}
* @throws NullPointerException if {@code workQueue}
*         or {@code threadFactory} is null
*/
public ThreadPoolExecutor(int corePoolSize,
                        int maximumPoolSize,
                        long keepAliveTime,
                        TimeUnit unit,
                        BlockingQueue<Runnable> workQueue,
                        ThreadFactory threadFactory) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
        threadFactory, defaultHandler);
}
```
如果直接向原生上述ThreadPoolExecutor的提交10000个任务，并且每个任务休眠5秒钟来模拟耗时操作，那么很快就会发生OOM。
>
由于瞬间提交的任务数非常大，且每个任务都耗时，导致按需创建/复用线程的策略基本上无法复用已经实例化的线程。在为新任务实例化线程时需要大量内存，因此OOM就再说难免了。

那么`OkHttp`是这样处理的么？

在前面的UML中，看到`Dispatcher`有三个Deque来存放任务，通过控制`maxRequests`和`maxRequestsPerHost`来限制最大并发数，和相主机同域名下的并发数。
默认的最大并发数是64,同域名并发数是5,支持个性化配置。

所以这里通过多个Deque来缓存尚未获得执行的任务，以及正在执行的任务，实现并发任务的调度。

## 双队列缓存监测

还记得在`RealCall`中执行任务时调用的一些接口么？

```java
client.dispatcher().finished(this);
```

在每个任务结束后，通过接口通知`Dispatcher`,再次检查任务队列，如果未触发最大并发数，则将新任务从等待队列已入执行队列。

```java
/** Used by {@code AsyncCall#run} to signal completion. */
void finished(AsyncCall call) {
    finished(runningAsyncCalls, call, true);
}

/** Used by {@code Call#execute} to signal completion. */
void finished(RealCall call) {
    finished(runningSyncCalls, call, false);
}

private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    Runnable idleCallback;
    synchronized (this) {
    if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
    if (promoteCalls) promoteCalls();
    runningCallsCount = runningCallsCount();
    idleCallback = this.idleCallback;
    }

    if (runningCallsCount == 0 && idleCallback != null) {
    idleCallback.run();
    }
}
```

具体移入操作由`promoteCalls`完成, 遍历`readyAsyncCalls`队列，加入`runningAsyncCalls`并提交给`executorService`:

```java
private void promoteCalls() {
    if (runningAsyncCalls.size() >= maxRequests) return; // Already running max capacity.
    if (readyAsyncCalls.isEmpty()) return; // No ready calls to promote.

    for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
    AsyncCall call = i.next();

    if (runningCallsForHost(call) < maxRequestsPerHost) {
        i.remove();
        runningAsyncCalls.add(call);
        executorService().execute(call);
    }

    if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
    }
}
```
