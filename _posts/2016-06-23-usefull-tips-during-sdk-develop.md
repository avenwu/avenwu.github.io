---
layout: post
title: "开发SDK需要考虑什么"
description: ""
header_image: /assets/img/2016-06-27-01.jpg
keywords: ""
tags: [android,sdk]
---
{% include JB/setup %}
![img](/assets/img/2016-06-27-01.jpg)

## 前言
做开发根据服务对象不同可以分为自有项目开发和对外的SDK开发，本文聊聊开发SDK的一些问题和解决思路。

## 目标和交付
工程开发是典型的输入 + 输出；开发之前需要明确需求，在这个过程中集中体现在如下几个环节的处理上：

	* 提供何种服务
	* 交付物料
	* 集成&使用

![sdk_overview]({{ site.baseurl }}/assets/images/sdk_overview.png)

### 提供何种服务
提供的服务指的是你的sdk解决什么问题，比如项umeng和sharesdk的解决三方分享的，facebook sdk解决FB登录的。
举例来说，一个公司，集团，随着业务线的扩展，出现了不同的小团队，每个团体研发的产品都需要接入集团的用户体系，在早期可能会每个团队根据业务需要各自接入账号，这样做技术上没什么么问题，问题在于：

	1. 每个团队都在做同样的事情，重复投入人力资源；
	2. 每个团队实现程度不同，安全性无法保障；
	3. 调整策略会导致每个接入方需要单独处理；

针对这种情况就有必要为账号体系提供一个统一的服务，业务方直接接入服务即可集成相关功能；
![sdk-service]({{ site.baseurl }}/assets/images/sdk-service.png)

### 资源交付
这个涉及到的比较宽泛，从物料形式上讲的话，你是对外体用jar包还是，资源文件，还是更进一步maven库依赖，集成的文档和对外的API文档。

* API文档要精简，只提供对外提供暴露的接口，内所有部实现则不需要提供，尽量做到透明化，接入方只需了解有限的服务接口即可。

* 利用javadoc我们可以生成sdk的api接口文档提供给接入方查阅，这就像java的api doc一样，可以查看相关的实现类，接口的方法等等包括他们的注释信息；

* 提供资源的时候尽量统一编码，比如使用UTF-8，输出doc的时候也制定编码格式，就可以保证不会出现中文乱码的诡异情况；

![sdk-resource]({{ site.baseurl }}/assets/images/sdk-resource.png)

## 如何集成
提供必要的接入文档和演示demo，有助于降低接入成本，但是还是难免会遇到一些尴尬的问题；
比如在聚合服务的时候，由于接入方的具体情况可能会出现三方库的冲突，比如分享库重复，如果是直接的原生的撒放文件冲突，那么使用一份即可，复杂一点的比如ShareSDK的jar包内部直接集成了sina的库中的实现文件，要解决这个问题，简单的办法是将ShareSDK的jar重新编辑，去除sina的实现文件，重新打包jar文件；

修改jar文件实际就是修改压缩包，java sdk提供了一个jar工具，通过jar工具我们可以直接修改具体jar包后重新打包：

* 查看jar内容
jar tf jar-file.jar

* 解压jar至当前目录
jar xf jar-file.jar

* 打包生成jar
jar cvf jar-file.jar input-file(s)

## 参考
* [http://docs.oracle.com/javase/tutorial/deployment/jar/](http://docs.oracle.com/javase/tutorial/deployment/jar/)
