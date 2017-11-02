---
layout: post
title: "写给Android开发者的CSS基础"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-07-26-02.jpg
keywords: "CSS"
tags: [前端]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-07-26-02.jpg)

## 前言
了解了简单HTML页面搭建后，再看看如何美化页面。

## 初见
CSS是专门给用于处理html样式的，有点像Android中style，可以为修饰页面的展示效果，其全称是Cascading Style Sheets。
## 掌握样式使用几种姿势
与Android中的style类似，写css也有几种方式；

* 作为单独的css文件，在html中引入；

```css
h1 {
	color:red;
}
```
* 在html内，写`<style></style>`，在标签内写样式；

```html
<style>
	h1 {
		color:red;
	}
</style>
```
* 在每个具体标签呢，通过inline的形式写style=""属性；

```html
<h1 style="color:red;">h1标题</h1>
```

## 样式和元素筛选
在html中的标签可以指定id，class，在css中定义对应的样式时有不同标示；

* id用#表示

```css
#redDiv {
	color:red;
}
```
* class用.表示

```css
.redClass {
	color:red;
}
```

当页面更复杂后，会产生大量标签嵌套的情况，这时定义标签的样式可以利用筛选的写法；

* 指定了class的标签,比如指定了class为red和link的a标签

```html
<a class="red link" href="https://www.baidu.com">baidu.com</>
```

```css
a.red {
	color:red;
}

.link {
	font-size:10px
}
```

* 指定嵌套标签下的的标签，比如div下直接嵌套的a标签

```html
<div>
	<a href="http://www.baidu.com">baidu.com</a>
</div>
```

```css
div > a {
	color:blue;
}
```
* 指定标签下的所有标签，如果div下所有的p标签,包含非直接嵌套的p

```html
<div>
    <p>content a</p>
    <a href="http://www.baidu.com"><p>content b</p></a>
</div>
```
```css
div p {
	color:blue;
}
```

* 根据标签属性选择

```html
<a href="http://www.baidu.com"><p>baidu.com</p></a>

```
```css
a[href="http://www.baidu.com"] {
  color: purple;
}
```

## 认识样式优先级
用chrome的inspect工具时经常会看到有很多css的设置诶标记的中划线，这些属性都是在优先级及比较厚被认为无效的属性设置，那么样式的属性设置优先级是如何确定的呢？

这个优先级遵循了几个原则：

* 命中元素，加一分
* 命中class, 加10分
* 命中id，加100分
* 内联的style，加1000分

在这些规则命中任然相同的情况下，已最后的设置位置，比如在css中靠后的相同属性设置覆盖前面的设置；

## 小结
css总体感觉对样式的操控，比较灵活，但是要尽量避免相同标签的样式重载，比较容易混淆视听，搞不清楚到底哪一处样式有效，虽然运行起来后可以借助工具看出来；
