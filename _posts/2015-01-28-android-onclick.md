---
layout: post
title: "android:onClick都做了什么"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-31.jpg
description: "从源码分析android:onClick实现机制"
category: "viewinject"
tags: [android]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-31.jpg)

相信大家都知道在layout里面可以给view写android:onClick属性，有没有好奇过它的内部是怎么实现的？  

## 前言
在用android:onClick的时候会有一些有意思的事情：
	
	比如说一般情况所在layout只能是Activity的，也就是说如果有一个Fragment对应的layout.xml,如果你在xml里写了android:onClick=“myClick”,同时在Fragment内实现public void myClick(View view),是会报错的。这是因为必须在Activity中声明该方法。
	
## 源码分析
找到android.view.View,可以发现这么一段代码：  

![view_onClick.png](http://7u2jir.com1.z0.glb.clouddn.com/view_onClick.png)
	 
代码比较好理解，首先解析出android:onClick的值，即获取方法名，然后通过反射，获取到Activity中对应的方法，并执行，如果找不对应方法则抛出异常。

* 为什么是Activity?
getContext().getClass()实际上view中的context都是其所在的Activity实例，那getClass之后当然就是在Activity中找

* 有什么用？
通过反射来访问方法其实是比较常见的，如果我们适当的加以利用那么也可以实现一定程度的代码配置，比如EventBus，中也有基于onEventXXX的方法声明约定，猜想也是利用这种方式实现的。