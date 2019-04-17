---
layout: post
title: "AndroidStudio小技巧--依赖库"
header_image: /assets/img/2016-03-06-26.jpg
description: "和eclipse中类似的project依赖配置"
category: 
tags: [工具]
---
{% include JB/setup %}
![img](/assets/img/2016-03-06-26.jpg)

## 前言
今天刚升级了AndroidStudio到1.1 RC 1,从其一年前刚推出的时候就果断从Eclipse转投AndroidStudio，总体来说选择是对的，虽然期间遇到过很多问题，但也正因为如此对AndroidStudio的很多配置有不少理解。

## 配置依赖项目

有时候我们会开发一些平台库项目，比如笔者写了一个support的Android库，用于记录这个理平时写的一些测试代码和自定义的东西，所以这个项目包含了sample和support两部分，现在我有另外一个项目A，也想开始依赖于support，怎么做比较合适。 

先来看以一下目录结构：

	Support
		|-sample
		|-support
	A Project
		|-app
		|-library
		

如果我已经将Support/support发布值maven，那么一切都没问题，直接用gradle添加依赖；但是由于support处于随时开发改变中，并不适合发布。

直接copy一份到A Project肯定是不行的，因为这样就存在两个副本要维护。


**解决办法就是手动配置依赖库的位置**：

	include ':app', ':library', ':support'
	project(':support').projectDir = new File(rootDir, "../support/support")
	
打开setting.gradle，包含support，然后指定其项目位置，我这里用的是相对路径。
剩下的就是在app的build.gradle里配置依赖了

	compile project(':support')

最后同步一下gradle，support会出现在左侧的导航面板中，就可以正常使用support中的资源了。

## 小结
这个方法相对来说既简单又实用，关键在于配置support的路径，这和Eclipse中的操作其实是类似的，只不过AndroidStudio目前并有有可视化的方法来添加目录并不在项目之内的库，所以需要自己手动配置。


