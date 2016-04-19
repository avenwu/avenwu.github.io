---
layout: post
title: "what do you know about image?"
description: "图片基础知识科普"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-04-18-02.jpg
keywords: "图片"
category: 
tags: [图片]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-04-18-02.jpg)

## 前言
好的图片搭配可以提升用户体验，本文一起看看开发中经常会打照面的各种图片

## 常见格式
图片根据二进制数据源是否采用有损压缩算法，可以分为无损格式（GIF）和有损格式（JPEG）

	* GIF (Graphic Information Format)
	* JPEG (Joint Photographic Experts Group)
	* PNG (Portable Network Graphic)
	* WebP ( NO2，Google)

## 图片构成
电子图片的核心是像素（Pixels）和宽高比（Aspect Ratio），由此一张图片根据宽高可以得出他的分辨率（Resolution）；  
早期的显示器一般都是4：3，现在变的更加的宽比如常见的都是16：9，16：10；实际上宽高比描述的是一张图面的形状，并不关心像素多少，所以在讲宽高比的时候也经常进行约分处理，16：10=》8：5。

现在我们知道图片由一个矩形的的像素构成，每个像素的色阶我们划分为256（2^8）, 根据每个像素用多少比特位（bit）表示，我们有了颜色深度（color depth）的概念，目前常见的8位，16位，24位，32位；
直观的理解，一个像素占的比特位越多，他所能表示的颜色范围也就越大，8位=》256，16位=>256 * 256, 24位=》256 * 256 * 256；
24位也叫作真彩（truecolor）,应为他已经表示了最大范围的色值（32位指的是ARGB，包括了alpha通道）；

## 图片大小
在了解了图片的组成之后，可以很方便的计算出图片元数据占用的空间（不考虑压缩算法带来的空间优化）；
以最常用的24位真彩(RGB)来说，100*100的图片占用（data footprint）的大小等于100 * 100 * 3 bytes = 100 * 100 * 3/1024 KB; 计算的时候需要注意单位，一般来说我们计算大小都是以字节的以上来衡量，并不会用比特位来算（啰嗦一句1bytes=8bits，一字节=8比特位），
	
	total size = width * height * 3 bytes

## 小结
本文是一片图片的科普小文，为学习图片的其他知识打好基础

## 参考
* Pro Android Graphic

