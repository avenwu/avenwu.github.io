---
layout: post
title: "编译fresco源码"
header_image: /assets/img/2016-03-06-17.jpg
description: ""
category: "fresco"
tags: [三方库]
---
{% include JB/setup %}
![img](/assets/img/2016-03-06-17.jpg)
## 前言
fresco出来已经有一阵子了，曾经尝试过一次clone源码编译，主要是看其自带的sample样例，但是除了一些错误，只能暂时搁置，今天再次想起这事，索性在来一遍，顺便分享一下遇到的问题即解决方案；

## Clone fresco
首先是获取代码，这个过程很快。

	git clone git@github.com:facebook/fresco.git
导入AndroidStudio也不是难事，只不过很多人都遇到了ndk-build的问题，问题在于机器上实际已经装了ndk,并且已经配置在path中，但死活就是编译不过;

## 解决编译问题
官网的说明中提到两点，一是ndk必须是10c以上的版本,二者需要手动配置ndk.path，注意不是ndk.dir虽然本质都是指向本地的ndk目录.

![/assets/build_prequisties.jpg](/assets/build_prequisties.jpg)

![/assets/ndk_path_gradle.jpg](/assets/ndk_path_gradle.jpg)

可以在用户目录下.gradle/中新建gradle.properties然后写上
	
	#osx/*nix
	ndk.path=/path/to/android_ndk/r10d

	#windows	
	ndk.path=C\:\\path\\to\\android_ndk\\r10d
笔者的用的是osx 10.10,设置后无效,依然报错.实际上这里变量的配置是为了让imagepipeline/build.gradle的正常执行，所以也可以像配置sdk.dir一样在项目中直接配置，写到local.properties或者gradle.properties中，再次编译通过。

## genymotion上无法部署
说也奇怪，居然不能在模拟器上跑，但build.gradle中实际上已经配置了arm/arm7/x86三种不同架构的flavor

	Unable to identify the apk for variant arm-debug and device genymotion-nexus_4___4_4_2___api_19___768x1280-192.168.56.101:5555
	
换真机正常跑起了sample样例

## 无数据空白页面
最后一步，app起来了，但是屏幕上只有一些参数，并没有想象中的图片加载.观察一下日志

![/assets/IMG_20150507_004726.JPG](/assets/IMG_20150507_004726.JPG)

```
05-07 00:45:53.280  30024-30189/com.facebook.fresco.sample E/unknown:FrescoSample﹕ Exception fetching album
    java.net.SocketTimeoutException: failed to connect to api.imgur.com/199.27.79.193 (port 443) after 15000ms
            at libcore.io.IoBridge.connectErrno(IoBridge.java:159)
            at libcore.io.IoBridge.connect(IoBridge.java:112)
            at java.net.PlainSocketImpl.connect(PlainSocketImpl.java:192)
            at java.net.PlainSocketImpl.connect(PlainSocketImpl.java:459)
            at java.net.Socket.connect(Socket.java:843)
            at com.android.okhttp.internal.Platform.connectSocket(Platform.java:131)
            at com.android.okhttp.Connection.connect(Connection.java:101)
            at com.android.okhttp.internal.http.HttpEngine.connect(HttpEngine.java:294)
            at com.android.okhttp.internal.http.HttpEngine.sendSocketRequest(HttpEngine.java:255)
            at com.android.okhttp.internal.http.HttpEngine.sendRequest(HttpEngine.java:206)
            at com.android.okhttp.internal.http.HttpURLConnectionImpl.execute(HttpURLConnectionImpl.java:345)
            at com.android.okhttp.internal.http.HttpURLConnectionImpl.connect(HttpURLConnectionImpl.java:89)
            at com.android.okhttp.internal.http.HttpsURLConnectionImpl.connect(HttpsURLConnectionImpl.java:161)
            at com.facebook.fresco.sample.urlsfetcher.ImageUrlsFetcher.downloadContentAsString(ImageUrlsFetcher.java:110)
            at com.facebook.fresco.sample.urlsfetcher.ImageUrlsFetcher.getImageUrls(ImageUrlsFetcher.java:75)
            at com.facebook.fresco.sample.urlsfetcher.ImageUrlsFetcher.access$000(ImageUrlsFetcher.java:41)
            at com.facebook.fresco.sample.urlsfetcher.ImageUrlsFetcher$1.doInBackground(ImageUrlsFetcher.java:63)
            at com.facebook.fresco.sample.urlsfetcher.ImageUrlsFetcher$1.doInBackground(ImageUrlsFetcher.java:60)
            at android.os.AsyncTask$2.call(AsyncTask.java:288)
            at java.util.concurrent.FutureTask.run(FutureTask.java:237)
            at android.os.AsyncTask$SerialExecutor$1.run(AsyncTask.java:231)
            at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1112)
            at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:587)
            at java.lang.Thread.run(Thread.java:841)
05-07 00:46:23.460  30024-30424/com.facebook.fresco.sample E/unknown:FrescoSample﹕ Exception fetching album
    java.net.SocketTimeoutException: failed to connect to api.imgur.com/199.27.79.193 (port 443) after 15000ms
            at libcore.io.IoBridge.connectErrno(IoBridge.java:159)
            at libcore.io.IoBridge.connect(IoBridge.java:112)
            at java.net.PlainSocketImpl.connect(PlainSocketImpl.java:192)
            at java.net.PlainSocketImpl.connect(PlainSocketImpl.java:459)
            at java.net.Socket.connect(Socket.java:843)
            at com.android.okhttp.internal.Platform.connectSocket(Platform.java:131)
            at com.android.okhttp.Connection.connect(Connection.java:101)
            at com.android.okhttp.internal.http.HttpEngine.connect(HttpEngine.java:294)
            at com.android.okhttp.internal.http.HttpEngine.sendSocketRequest(HttpEngine.java:255)
            at com.android.okhttp.internal.http.HttpEngine.sendRequest(HttpEngine.java:206)
            at com.android.okhttp.internal.http.HttpURLConnectionImpl.execute(HttpURLConnectionImpl.java:345)
            at com.android.okhttp.internal.http.HttpURLConnectionImpl.connect(HttpURLConnectionImpl.java:89)
            at com.android.okhttp.internal.http.HttpsURLConnectionImpl.connect(HttpsURLConnectionImpl.java:161)
            at com.facebook.fresco.sample.urlsfetcher.ImageUrlsFetcher.downloadContentAsString(ImageUrlsFetcher.java:110)
            at com.facebook.fresco.sample.urlsfetcher.ImageUrlsFetcher.getImageUrls(ImageUrlsFetcher.java:75)
            at com.facebook.fresco.sample.urlsfetcher.ImageUrlsFetcher.access$000(ImageUrlsFetcher.java:41)
            at com.facebook.fresco.sample.urlsfetcher.ImageUrlsFetcher$1.doInBackground(ImageUrlsFetcher.java:63)
            at com.facebook.fresco.sample.urlsfetcher.ImageUrlsFetcher$1.doInBackground(ImageUrlsFetcher.java:60)
            at android.os.AsyncTask$2.call(AsyncTask.java:288)
            at java.util.concurrent.FutureTask.run(FutureTask.java:237)
            at android.os.AsyncTask$SerialExecutor$1.run(AsyncTask.java:231)
            at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1112)
            at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:587)
            at java.lang.Thread.run(Thread.java:841)
```

这个原因比较明显，我大天朝，外网不是想上就能上的。开启vpn翻墙，退出app，再次进入，开始加载图片

![/assets/IMG_20150507_004926.JPG](/assets/IMG_20150507_004926.JPG)
