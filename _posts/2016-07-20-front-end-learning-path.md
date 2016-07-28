---
layout: post
title: "点亮前端学习的技能树"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-07-28-01.jpg
keywords: ""
tags: [前端]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-07-28-01.jpg)

## 前言
不会前端的Android工程师不是合格的码农；

## 前端技能树
汇总前端学习过程中的心得笔记；

{% assign front_end_tags = site.front_end_tags | split: ", " %}
{% for tag in front_end_tags %} 
<h4>{{ tag }}</h4>
<ul>
{% assign pages_list = site.tags[tag] %}  
{% include JB/pages_list %}
</ul>
{% endfor %}