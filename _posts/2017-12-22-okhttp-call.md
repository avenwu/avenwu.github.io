---
layout: post
title: "OkHttp【三】Call/RealCall源码分析"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2017-12-22-01.jpg
keywords: "okhttp call"
tags: [okhttp]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2017-12-22-01.jpg)

## 前言

在`OkHttpClient`实例化一个请求时，我们使用了`newCall`方法来构造一个`Call`对象，并执行。
本文一起分析`Call`的相关实现逻辑。

```java
/**
* Prepares the {@code request} to be executed at some point in the future.
*/
@Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
}
```

## Call 的作用

`Call`是一个接口，具体实现类为`RealCall`, 可以从接口定义了解其作用和定位。

```java
public interface Call extends Cloneable {
  Request request();

  Response execute() throws IOException;

  void enqueue(Callback responseCallback);

  void cancel();

  boolean isExecuted();

  boolean isCanceled();

  Call clone();

  interface Factory {
    Call newCall(Request request);
  }
}
```

原代码注释很全，简单理解就是：`Call`是准备好可以执行的请求，允许被取消，但是一个`Call`只能执行一次。

主要的几个方法如下：

* request()
由于请求参数配置是通过`Request`来封装，因此`Call`会持有`Request`对象实例，并提供get方法。
* execute()
同步执行请求，并返回`Response`
* enqueue()
异步执行请求，并将结果通过回调接口返回
* cancel()
取消请求，不一定可靠，已经开始的请求时中断不了的

## RealCall 的实现

`RealCall`毋庸置疑，实现了`Call`接口，并且会持有一些成员：
* OkHttpClient
用于分发请求
* RetryAndFollowUpInterceptor
用于重试
* EventListener
消息监听器，这个比较恶心，和Call相互依赖,具体代码在`newRealCall`的实现

```java
  static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.eventListener = client.eventListenerFactory().create(call);
    return call;
  }
```

## 同步execute

整个`RealCall`的实现主要在于同步执行和异步执行两个方法，先看下同步`execute`：

```java
  @Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    try {
      client.dispatcher().executed(this);
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      eventListener.callFailed(this, e);
      throw e;
    } finally {
      client.dispatcher().finished(this);
    }
  }
```

可以看到，同步执行请求时，会做一下判断，重复执行直接抛异常。整个过程通过`eventListener`来通知状态，比如`callStart`,`callFailed`,`finished`。

`Response`的返回放在了`getResponseWithInterceptorChain`方法中，我们知道OkHttp允许我们配置各种拦截器，就是利用了这个链式拦截器来实现的。

即使我们没有主动配置任何拦截器，也会有至少有内置的5个拦截器。针对这几个拦截器的实现，后续我们在单独分析。

现在我们已经知道要返回一个`Response`需要经过很多链式调用，负责链式调用的类就是`RealInterceptorChain`，这个类内部会为每个拦截器递归实例化一个`RealInterceptorChain`，并执行响应的方法。

```java
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```

注意一个`RealInterceptorChain`的proceed是会被调用者执行的，第一次调用是在`RealCall`中，后续则是每次被拦截器的具体实现类调用`Interceptor#intercept`.
最后一个拦截器是`CallServerInterceptor`,看名字就可以猜到他是真正执行网络请求的类，前面的一些都只是加工，比如缓存处理，连接池等等。

## 链式调用拦截器

在遍历每个拦截器时都会专门创建一个`RealInterceptorChain`，因此我们来看下这个Chain都会做些什么事情？
通过阅读`proceed`方法可以看到它主要是在遍历拦截器，为每个拦截器实例化一个独立的Chain对象，同时根据遍历情况，做了大量的异常抛出，这样可以使非预期的执行被暴露出来。

```java
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    // If we already have a stream, confirm that the incoming request will use it.
    if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed().
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    if (response.body() == null) {
      throw new IllegalStateException(
          "interceptor " + interceptor + " returned a response with no body");
    }

    return response;
  }
```

上述实现中最核心的就是递归实例化，形成了链式依赖，每一个拦截器的返回都会需要下一个拦截器的的返回结果。
在这里每个Chain都是通过proceed返回Response，在proceed中调用下一个拦截器的intercept方法，而intercept的实现中又必然会调用Chain的proceed方法，直到最后一个链。

`RealInterceptorChain#proceed`

```java
// Call the next interceptor in the chain.
RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
    connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
    writeTimeout);
Interceptor interceptor = interceptors.get(index);
Response response = interceptor.intercept(next);
```

`RetryAndFollowUpInterceptor#intercept`

```java
@Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Call call = realChain.call();
    // ...
    while (true) {
      if (canceled) {
        streamAllocation.release();
        throw new IOException("Canceled");
      }

      Response response;
      boolean releaseConnection = true;
      try {
        response = realChain.proceed(request, streamAllocation, null, null);
        releaseConnection = false;
      } catch (RouteException e) {
        // The attempt to connect via a route failed. The request will not have been sent.
        if (!recover(e.getLastConnectException(), false, request)) {
          throw e.getLastConnectException();
        }
        releaseConnection = false;
        continue;
      }
      // ...
      }
      // ...
    }
  }

```

## 异步enqueue

现在简单分析下异步调用，可以看到他们的主要差异在于，执行call会被丢到一个队列中，由`OkHttpClient`的分发器进行调用。

```java
  @Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```

可以推测`AsynCall`内部会有和同步执行类似调用关系, 这里的`AsynCall`实际上就是一个`Runnable`，只是做了一下命名和简单包装`execute`方法

```java
  final class AsyncCall extends NamedRunnable {
    private final Callback responseCallback;

    AsyncCall(Callback responseCallback) {
      super("OkHttp %s", redactedUrl());
      this.responseCallback = responseCallback;
    }
    // ...

    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          eventListener.callFailed(RealCall.this, e);
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
  }
```

在`execute`中，还是通过前面分析的`getResponseWithInterceptorChain`来返回`Response`,除此之外就是`eventListener`的消息回调，已经最后的响应回调。
可以看到这里的响应回调就是直接在当前线程执行了`Callback`的方法，因此不存在线程调度问题，因为`OkHttp`是可以给纯Java项目使用的，对线程的调度应该在其他地方。
关于这一点后续我们可以做实验来验证一下。
