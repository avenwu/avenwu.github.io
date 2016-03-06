---
layout: post
title: "Android研究系列"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-41.jpg
tagline: ""
tags : [android]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-41.jpg)

## 前言
自12年从事Android软件开发至今，接触了很多人与事，也学习到了很多。  
为了能更深入的掌握Android的相关技术，列了一个大纲，比如了解一些轮子的设计和实现，必要的轮子，还是得自己造一把。

## 视图方向
	
* 【控件】下拉刷新
	
{%for post in site.categories.pulltorefresh%}	
	[{{post.title}}]({{ site.baseurl }}{{ post.url }})
{% endfor %}
	
* 【控件】滑动删除
	
{%for post in site.categories.slidedelete%}	
	[{{post.title}}]({{ site.baseurl }}{{ post.url }})
{% endfor %}
	
* 【控件】自定义ViewGroup
	
{%for post in site.categories.customlayout%}	
	[{{post.title}}]({{ site.baseurl }}{{ post.url }})
{% endfor %}
	
* 【绘制】View绘制
	
{%for post in site.categories.viewdraw%}	
	[{{post.title}}]({{ site.baseurl }}{{ post.url }})
{% endfor %}
	
* 【事件】touch事件
	
{%for post in site.categories.touchevent%}	
	[{{post.title}}]({{ site.baseurl }}{{ post.url }})
{% endfor %}
	
* 【动画】View动画
	
{%for post in site.categories.viewanimation%}	
	[{{post.title}}]({{ site.baseurl }}{{ post.url }})
{% endfor %}
	
* 【动画】ViewGroup容器类动画
	
{%for post in site.categories.layoutanimation%}	
	[{{post.title}}]({{ site.baseurl }}{{ post.url }})
{% endfor %}
	

## 组件解耦
* 【注入】视图注入

{%for post in site.categories.viewinject%}	
	[{{post.title}}]({{ site.baseurl }}{{ post.url }})
{% endfor %}

* 【注入】事件注入
* 【注入】动态代码生成

{%for post in site.categories.annotation%}	
	[{{post.title}}]({{ site.baseurl }}{{ post.url }})
{% endfor %}

* 【 IOC 】xml配置
* 【实例】EventBus

{%for post in site.categories.ioc%}	
	[{{post.title}}]({{ site.baseurl }}{{ post.url }})
{% endfor %}

* 【系统】ContentProvider自定义


## 网络通信
* 【实例】Retrofit
* 【实例】Volley
* 【协议】TCP/IP通信
* 【协议】HTTP实现
* 【协议】WebSocket实现


## 缓存技术
* 【图片】Picaso实现
* 【图片】Volley实现
* 【图片】Universal Image Loader实现
* 【策略】LruCache分析
* 【策略】DiskLruCache分析