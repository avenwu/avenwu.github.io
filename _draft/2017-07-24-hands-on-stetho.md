---
layout: post
title: "Stetho上手体验"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-04-18-02.jpg
keywords: ""
tags: []
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-04-18-02.jpg)

# Stetho

这个调试工具其实已经除了很久了，不过一直没有用过，最近接入debug体验了下，果然非常的直观；
* View Hierarchy
* data,db
* Network Viewer

这些功能长期以来都可以利用Android SDK提供的ABD, hierarchy，Log的实现，Stetho把这些都做成了可视化的Chrome Dev Tools，其他功能到是没什么差异，不过可视化查看数据这一点着实很方便；

```
61.91.161.217 chrome-devtools-frontend.appspot.com
61.91.161.217 chrometophone.appspot.com
```

# 参考

* [https://www.littlerobots.nl/blog/stetho-for-android-debug-builds-only/](https://www.littlerobots.nl/blog/stetho-for-android-debug-builds-only/)
* [https://github.com/facebook/stetho](https://github.com/facebook/stetho)
