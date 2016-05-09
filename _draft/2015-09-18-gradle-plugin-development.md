---
layout: post
title: "gradle插件开发"
description: ""
category: 
tags: [gradle]
---
{% include JB/setup %}

## 前言
这是一篇迟到的笔记，15年已经创建草稿，但是知道今天才真正的动笔，是在惭愧

## 热身知识
一般写gradle插件都是为了构建项目，提供一些小功能，在整个插件开发中Project和Task是两个最重要的概念；  
project比较好理解，就是工程主体，他包含一些基本的属性，task是project内定义的实际干活的对象，比如clean，assembleDebug都是task；  
gradle插件开发可以使用java，groovy，scala等语言，他们都是基于jvm的语言；

## 写一个Hello world插件
目前来说插件可以直接在build.gradle内实现，也可以独立到单独文件实现，根据官方教程，我们分别开一下两种实现；  

先来看一下直接在build.gradle内的实现：
创建build.gradle文件

	touch build.gradle

用你习惯的任意文本编辑器打开build.gradle，敲入下面的groovy代码

{% highlight groovy %}
apply plugin: GreetingPlugin

class GreetingPlugin implements Plugin<Project> {
    void apply(Project project) {
    	project.extensions.create("e", YourExtension)
        project.task('hello') << {
        	project.e.text = 'Hello from the GreetingPlugin'
        	println project.e.text
        }
    }
}
class YourExtension {
	def String text = "Default text"
}
{% endhighlight %}

这段代码实际上实现了Plugin, 自定义了名为GreetingPlugin的插件，同时定义了一个名为YourExtension的model，在Plugin中我们定义了名为hello的task，并在task中执行一段输出操作；  
	
	gradle hello


{% highlight bash %}
avens-MacBook-Pro:hello-world aven$ gradle hello
:hello
Hello from the GreetingPlugin

BUILD SUCCESSFUL

Total time: 4.379 secs

This build could be faster, please consider using the Gradle Daemon: https://docs.gradle.org/2.7/userguide/gradle_daemon.html
{% endhighlight %}

一个最基本的gradle插件实际上就完成了，但是如果我们需要发布这个插件，并在其他工程中引用要怎么做呢？  
下面我们把插件实现剥离出来： 

## 参考
[https://docs.gradle.org/current/userguide/custom_plugins.html](https://docs.gradle.org/current/userguide/custom_plugins.html)

[https://plugins.gradle.org/docs/publish-plugin](https://plugins.gradle.org/docs/publish-plugin)