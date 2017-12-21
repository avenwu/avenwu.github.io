---
layout: post
title: "OkHttpClient 源码分析"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2017-12-21-03.jpg
keywords: "okhttp okhttpclient"
tags: [okhttp]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2017-12-21-03.jpg)

## 背景

回顾一下，在最简单的`GET`请求场景中，我们的程序都做了那些事情？

在不考虑个性化配置的情况下，我们只需三步：

1. 我们首先实例化了一个`OkHttpClient`，如果有多个请求需要发送，这个类将会做一个单例来复用；
2. 同时我们会构造一个`Request`实例，用于传入我们的请求参数；
3. 最后通过`OkHttpCLient`实例，`new`出一个`Call`, 并执行(或者丢入异步队列)。

```java
OkHttpClient client = new OkHttpClient();

String run(String url) throws IOException {
  Request request = new Request.Builder()
      .url(url)
      .build();

  Response response = client.newCall(request).execute();
  return response.body().string();
}
```
本文一起分析`OkHttpClient`的实现。

## OkHttpCLient 构造

关于`OkHttpClient`提供的API，可以参考源代码，也可以在线查询[Java doc](http://square.github.io/okhttp/3.x/okhttp/)文档。

`OkHttpClient`用于构造请求的`Call`，从资源开销和复用角度来说，一个管理类一般都是左侧实例复用。比如全局单例，或者由使用者自行构造为静态变量，实例复用。
因此`OkHttpClient`页被建议尽可能实例共享，做复用。

构造`OkHttpClient`既可以直接`new`出来：

```java
// The singleton HTTP client.
public final OkHttpClient client = new OkHttpClient();
```

当然更多时候，我们是通过`OkHttpClient.Builder`来创建实例，这样可以方便的进行配置，比如拦截器，日志，缓存等设置：

```java
// The singleton HTTP client.
public final OkHttpClient client = new OkHttpClient.Builder()
    .addInterceptor(new HttpLoggingInterceptor())
    .cache(new Cache(cacheDir, cacheSize))
    .build();
```

上述两种方式创建出来的`OkHttpClient`都是一个完全独立的实例，其内部有独立的链接池，线程池，配置信息。

如果我们想创建一个`OkHttpClient`，并且使其复用原有实例的这些链接池，线程池，配置信息，如何处理呢？

```java
OkHttpClient eagerClient = client.newBuilder()
    .readTimeout(500, TimeUnit.MILLISECONDS)
    .build();
Response response = eagerClient.newCall(request).execute();
```

如上，可以看到`OkHttpClient`提供了一个`newBuilder`的方法，来方便我们进行共享配置；复用的思想是通过构造器赋值，我们来看下具体实现代码：

*OkHttpClient.java*

```java
  public Builder newBuilder() {
    return new Builder(this);
  }
```

通过`newBuilder`实例化`Builder`对象时，我们传入了当前的`OkHttpClient`对象，然后在`Builder`的有参构造器中初始化:

```java
// 无参构造，独立的线程池，连接池等
public Builder() {
    dispatcher = new Dispatcher();
    protocols = DEFAULT_PROTOCOLS;
    connectionSpecs = DEFAULT_CONNECTION_SPECS;
    eventListenerFactory = EventListener.factory(EventListener.NONE);
    proxySelector = ProxySelector.getDefault();
    cookieJar = CookieJar.NO_COOKIES;
    socketFactory = SocketFactory.getDefault();
    hostnameVerifier = OkHostnameVerifier.INSTANCE;
    certificatePinner = CertificatePinner.DEFAULT;
    proxyAuthenticator = Authenticator.NONE;
    authenticator = Authenticator.NONE;
    connectionPool = new ConnectionPool();
    dns = Dns.SYSTEM;
    followSslRedirects = true;
    followRedirects = true;
    retryOnConnectionFailure = true;
    connectTimeout = 10_000;
    readTimeout = 10_000;
    writeTimeout = 10_000;
    pingInterval = 0;
}
// 有参构造，拷贝传入的实例配置
Builder(OkHttpClient okHttpClient) {
    this.dispatcher = okHttpClient.dispatcher;
    this.proxy = okHttpClient.proxy;
    this.protocols = okHttpClient.protocols;
    this.connectionSpecs = okHttpClient.connectionSpecs;
    this.interceptors.addAll(okHttpClient.interceptors);
    this.networkInterceptors.addAll(okHttpClient.networkInterceptors);
    this.eventListenerFactory = okHttpClient.eventListenerFactory;
    this.proxySelector = okHttpClient.proxySelector;
    this.cookieJar = okHttpClient.cookieJar;
    this.internalCache = okHttpClient.internalCache;
    this.cache = okHttpClient.cache;
    this.socketFactory = okHttpClient.socketFactory;
    this.sslSocketFactory = okHttpClient.sslSocketFactory;
    this.certificateChainCleaner = okHttpClient.certificateChainCleaner;
    this.hostnameVerifier = okHttpClient.hostnameVerifier;
    this.certificatePinner = okHttpClient.certificatePinner;
    this.proxyAuthenticator = okHttpClient.proxyAuthenticator;
    this.authenticator = okHttpClient.authenticator;
    this.connectionPool = okHttpClient.connectionPool;
    this.dns = okHttpClient.dns;
    this.followSslRedirects = okHttpClient.followSslRedirects;
    this.followRedirects = okHttpClient.followRedirects;
    this.retryOnConnectionFailure = okHttpClient.retryOnConnectionFailure;
    this.connectTimeout = okHttpClient.connectTimeout;
    this.readTimeout = okHttpClient.readTimeout;
    this.writeTimeout = okHttpClient.writeTimeout;
    this.pingInterval = okHttpClient.pingInterval;
}
```

有没有觉得这个模式非常熟悉？

其实在`Android`系统框架中，大量存在类似的思想，比如`Handler`的构造，默认是绑到当前线程上，共享一个`Looper`，但是我们也可以配置到独立的线程和Looper，从而实现异步线程上的消息队列处理。

## OkHttpClient 接口

整个`OkHttpClient`的功能接口只有3个：

* 构造新的Builder
* 构造Call
* 构造WebSocket

其他的接口都是getter，获取配置信息：

```java
  final Dispatcher dispatcher;
  final @Nullable Proxy proxy;
  final List<Protocol> protocols;
  final List<ConnectionSpec> connectionSpecs;
  final List<Interceptor> interceptors;
  final List<Interceptor> networkInterceptors;
  final EventListener.Factory eventListenerFactory;
  final ProxySelector proxySelector;
  final CookieJar cookieJar;
  final @Nullable Cache cache;
  final @Nullable InternalCache internalCache;
  final SocketFactory socketFactory;
  final @Nullable SSLSocketFactory sslSocketFactory;
  final @Nullable CertificateChainCleaner certificateChainCleaner;
  final HostnameVerifier hostnameVerifier;
  final CertificatePinner certificatePinner;
  final Authenticator proxyAuthenticator;
  final Authenticator authenticator;
  final ConnectionPool connectionPool;
  final Dns dns;
  final boolean followSslRedirects;
  final boolean followRedirects;
  final boolean retryOnConnectionFailure;
  final int connectTimeout;
  final int readTimeout;
  final int writeTimeout;
  final int pingInterval;
```