---
layout: post
title: "OkHttp 开篇"
description: "OkHttp源码分析"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2017-12-20-02.jpg
keywords: "OkHttp"
tags: [OkHttp]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2017-12-20-02.jpg)

## 前言

软件开发领域在不断产生新技术，新框架。作为Android开发者，我们经历了他的巨大演进，从2.1到如今的8.0，系统本身在变化，开源世界也在变化。
过去每个团队在使用自身封装的图片框架，网络框架，如今`Picasso`，`Fresco`已经逐步取而代之，甚至Android系统本身也内置了`OkHttp`的支持。

站在使用者的角度，到最后也只能是使用者，如果要更好理解这些框架背后的技术原理，有必要对其进行深度的源码剖析。

后续分析，将基于如下版本：

* OkHttp: [http://square.github.io/okhttp/](http://square.github.io/okhttp/)
* 版本：3.9.1

```groovy
compile 'com.squareup.okhttp3:okhttp:3.9.1'
```

## OkHttp杂谈

整个`OkHttp`发展非常迅猛，撰写本文的时候最新的发行版本为`3.9.1`. 官方为了区分几个大版本采用了不同的group id与包名，因此迁移不同版本还有有些工作要做的。

在笔者印象中，`OkHttp`本身源自`Square`开发的`Retrofit`，而后其中的网络模块逐步独立演变为了单独的项目`OkHttp`。然而无论有什么渊源，这并不影响我们现在对`OKHttp`做分析/解读。

目前`OkHttp`托管于GitHub：[https://github.com/square/okhttp](https://github.com/square/okhttp)

```bash
git clone git@github.com:square/okhttp.git
```

根据可追溯的历史，`v1.0.0`发布于[2013/05/06](https://github.com/square/okhttp/commit/d95ecff5423f19e019e178baddfe4211f2fe57aa)，作者是顶顶大名的[JakeWharton](https://github.com/JakeWharton)

Jake是位多产的大神,感受下他的全年Github提交记录：

![jakewharton-commits]({{ site.baseurl }}/assets/images/jakewharton-commits.png)

同时开发了其他诸多项目：

![jakewharton-repo]({{ site.baseurl }}/assets/images/jakewharton-repo.png)

## OkHttp定位

OkHttp用于Android/Java程序进行Http、Http/2网络通信，因此可以在基于Java的项目中使用。（Android 2.3+, Java 1.7+）

HTTP是现代网络通信常用协议，高效进行HTTP通信可以使我们的程序加载更快速，并且节省带宽。

对OkHttp来说他有以下几个优点：

* 支持HTTP/2，允许所有相同host的请求复用一个socket。
* 连接池降低请求延迟（如果HTTP/2不可用）。
* 透明的GZIP支持，压缩下载体积。
* 支持响应体的缓存，避免重复请求。

另外，OkHttp在网络不良的情况下，他能够静默从常见连接问题中恢复，支持多IP自动重试等。
最后要提的一点是，使用简单。他的`请求/响应`式的API接口是通过Builder构造器来实现的，支持同步或异步的请求调用

## OkHttp示例

进行OkHttp的演示前，根据自己的实际情况，进行依赖库的配置.

MAVEN

```xml
<dependency>
  <groupId>com.squareup.okhttp3</groupId>
  <artifactId>okhttp</artifactId>
  <version>3.9.1</version>
</dependency>
```

GRADLE

```groovy
compile 'com.squareup.okhttp3:okhttp:3.9.1'
```

* 发起一个GET请求

下面的示例将访问一个url，并且把请求的响应体以文本形式同步返回。

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

* 发起一个POST请求

这个例子中，我们将向服务器发送一个POST请求，并上传一段JSON数据作为请求体，最后同步返回请求的响应体。

```java
public static final MediaType JSON
    = MediaType.parse("application/json; charset=utf-8");

OkHttpClient client = new OkHttpClient();

String post(String url, String json) throws IOException {
  RequestBody body = RequestBody.create(JSON, json);
  Request request = new Request.Builder()
      .url(url)
      .post(body)
      .build();
  Response response = client.newCall(request).execute();
  return response.body().string();
}
```

一个最简单的HTTP请求就完成了，在实际项目中，我们的情景会更加复杂一些，比如GET，POST参数，HEADER处理，请求拦截等等。
因此后续我们将会进一步深入分析，挖掘OkHttp的更多细节。

