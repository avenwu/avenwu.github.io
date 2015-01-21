---
layout: post
title: "Android研究系列"
tagline: ""
tags : [android]
---
{% include JB/setup %}

###视图方向

* 【控件】侧拉菜单
	
{%for post in site.categories.slidingmenu%}	
	[{{post.title}}]({{ site.baseurl }}{{ post.url }})
{% endfor %}
	
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
	

###组件解耦
* 【注入】视图注入
* 【注入】事件注入
* 【注入】动态代码生成
* 【 IOC 】xml配置
* 【实例】EventBus
* 【实例】Otto
* 【系统】ContentProvider自定义


### 网络通信
* 【实例】Retrofit
* 【实例】Volley
* 【协议】TCP/IP通信
* 【协议】HTTP实现
* 【协议】WebSocket实现


### 缓存技术
* 【图片】Picaso实现
* 【图片】Volley实现
* 【图片】Universal Image Loader实现
* 【策略】LruCache分析
* 【策略】DiskLruCache分析