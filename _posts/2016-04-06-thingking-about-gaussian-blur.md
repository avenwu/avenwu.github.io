---
layout: post
title: "高斯平滑是怎么回事"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-04-07-01.jpg
keywords: "Gaussian blur 高斯模糊"
category: 
tags: [算法]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-04-07-01.jpg)

## 前言
	
	算法之重要，不为牛逼，更在其解决问题的实用价值

最近正好看到很久以前收藏的[代震军][1]大牛写的图像处理效果，洋洋洒洒几十种像素处理，实在值得我们这些晚辈好好学习；

## 高斯平滑/模糊
在Android中如果做模糊效果，目前性能最好的方案是用RenderScript实现的Gaussian Blur，也就是高斯模糊，也叫作高斯平滑；原谅我之前不知道高斯模糊的原理， google大法好，立刻发掘出了一把相关的数学原理介绍，比较好的有[阮一峰][2]的一篇科普文章，当然不能漏了维基百科上的介绍[Gaussian Blur][3];

### Gaussian原理是什么？
高斯平滑，实际上就是对图片内所有的像素点的一种平滑处理，视觉效果上直观的感受就是图片的降噪和细节消除，这个应该不难理解；

降噪是什么？举个简单例子，用低像素手机拍的照片，特别是灯光不好的环境，拍出来的照片容易有很多亮点，色块，降噪就是把这些东西去除；看看这张维基上的配图

![https://upload.wikimedia.org/wikipedia/commons/thumb/8/87/Highimgnoise.jpg/300px-Highimgnoise.jpg](https://upload.wikimedia.org/wikipedia/commons/thumb/8/87/Highimgnoise.jpg/300px-Highimgnoise.jpg)

细节消除就是字面意思，我们希望干净利落的细节都不要，怎么去掉呢，就是通过像素点周边像素色值去一个均值，这个均值是一种加权均值，根据周边元素的距离来确定权重；  
为什么这么做呢？我们模糊希望的是相邻的色值趋于融合，而里的很远的像素之间其实是没什么关系的，所以这个权重在像素位置中的关系就变得很奇妙，他的性质实际就是一个正态分布了；还看一张维基的配图，是不是想起了大学的高等数学了呢？

![https://upload.wikimedia.org/wikipedia/commons/thumb/7/74/Normal_Distribution_PDF.svg/360px-Normal_Distribution_PDF.svg.png](https://upload.wikimedia.org/wikipedia/commons/thumb/7/74/Normal_Distribution_PDF.svg/360px-Normal_Distribution_PDF.svg.png)

### 高斯函数
下面重要的来了(抱紧我也没用)，既然是加权求值，也知道了是正态分布，那你告诉到底如何计算？公式来了，拿去不谢，看不看得懂就看还给老师多少了：）
一维  
![http://homepages.inf.ed.ac.uk/rbf/HIPR2/eqns/eqngaus1.gif](http://homepages.inf.ed.ac.uk/rbf/HIPR2/eqns/eqngaus1.gif)

二维  
![http://homepages.inf.ed.ac.uk/rbf/HIPR2/eqns/eqngaus2.gif](http://homepages.inf.ed.ac.uk/rbf/HIPR2/eqns/eqngaus2.gif)

图片处理用的是二维的公式；

这一块可以参考[阮一峰][2]文章的相关介绍；

。。。休息一下，稍后更精彩！

### 理解高斯函数
公式很简单，但是理解还得花点功夫。  

### 图片和像素
平常我们说的图片实际上一般指的是他在媒介上的视觉呈现，比如电子显示屏，书籍，在这里我们谈的高斯平滑直接处理的是图片的二进制数据；  
一张图片本身是有无数的像素点构成，比如常见的二维平面下，一张图片构成可以理解为如下图：

![image-digital-structure.jpg]({{ site.baseurl }}/assets/images/image-digital-structure.jpg)

现在结合我们知道的二维高斯公式和图片的像素表示，通过函数求导出一个像素点周边每个临近像素的权重，可以得到类似一个表，再将这个表和像素元素相乘，求和, 还是看一下大牛画的图：

![http://image.beekka.com/blog/201211/bg2012111412.png](http://image.beekka.com/blog/201211/bg2012111412.png)

![http://image.beekka.com/blog/201211/bg2012111413.png](http://image.beekka.com/blog/201211/bg2012111413.png)

![http://image.beekka.com/blog/201211/bg2012111414.png](http://image.beekka.com/blog/201211/bg2012111414.png)

## 小结
基本上高斯模糊和原理就是利用正态分布，对每个像素镜像处理，在实际的算法实现中，由于完美高斯的复杂度开销其实很高，因此在效果相差很小的情况下也出现了很多简化版的算法实现;
管高斯算法的实现，下次再见：）


## 参考
* [http://www.ruanyifeng.com/blog/2012/11/gaussian_blur.html](http://www.ruanyifeng.com/blog/2012/11/gaussian_blur.html)
* [http://www.cs.cornell.edu/courses/cs6670/2011sp/lectures/lec02_filter.pdf](http://www.cs.cornell.edu/courses/cs6670/2011sp/lectures/lec02_filter.pdf)
* [https://en.wikipedia.org/wiki/Gaussian_blur](https://en.wikipedia.org/wiki/Gaussian_blur)

	[1]:http://www.cnblogs.com/daizhj/
	[2]:http://www.ruanyifeng.com/blog/2012/11/gaussian_blur.html
	[3]:https://en.wikipedia.org/wiki/Gaussian_blur