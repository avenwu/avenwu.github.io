---
layout: post
title: "写给Android开发者的HTML基础"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-07-26-01.jpg
keywords: "HTML"
tags: [前端, HTML]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-07-26-01.jpg)

## 前言
随着全栈工程师的兴起，工作之外掌握其他的开发技术变得越来越重要；本文简单讲介绍一些HTML里的必知必会知识点。

## HTML页面构成
一个html页面的结构如下：

```html
<!DOCTYPE html>
<html>
	<head></head>
	<body></body>
</html>
```
第一行是文件标识，代表页面是html；第一个标签是html，head中是一些头信息，比如title，link等，body中放的是具体的页面元素。

## HTML常用标签及含义
作为标记语言，html有一些原生的标签需要掌握，这类似于其他任何GUI编程中的UIKit，比如Andorid中TextView，FrameLayout等等；

* `<h1></h1>,h2...h6`标题标签
* `<p></p>`段落标签
* `<span></span>`样式标签，用于在一段文本之内指定某一节文本的样式，类似于andorid中Spannable
* `<a></a>`超链，超链的可以包裹一个div使得整个div点击能跳转

```html
<a href="https://avenwu.net">
	<div style="width:100px;height=100px;background-color:red;">
		<p>这是一段文字</p>
	</div>
</a>
```

* `<img></img>` 图片标签
* `<ol></ol>` 排序列表标签，ordered list
* `<ul></ul>` 无序列表标签，unordered list
* `<li></li>` 列表标签条目，list item

```html
<ol>
	<li>A</li>
	<li>B</li>
	<li>C</li>
</ol>

<ul>
	<li>张三</li>
	<li>李四</li>
	<li>王五</li>
</ul>
```

* `<strong></strong>`强调，会加粗文字
* `<em></em>`重要，会使文字斜体
* `<!-- 被注释内容 -->`注释
* `<table></table>`表格
* `<tr></tr>`行 table row
* `<td></td>`单元格,table data
* `<thead></thead>`表头
* `<th></th>`表头行
* `<tbody></tbody>`表格body
* `<div></div>` divide区域

## 常用属性
* font-size字号
* font-family 字体
* color颜色
* background-color背景色
* text-align文本对齐
* float对齐方式
* clear和float经常上下结合使用

## 小结
html还有很多其他常用表情，这里没有一一列举；
