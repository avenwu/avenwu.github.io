---
layout: post
title: "android打电话避免添加权限"
header_image: /assets/img/2016-03-06-14.jpg
description: "android打电话避免添加权限"
category: 
tags: [android]
---
{% include JB/setup %}
![img](/assets/img/2016-03-06-14.jpg)

## 前言
国内的Andorid应用不管是什么类型的，动不动就要求读取通讯录，打电话等等敏感权限，非常恶心；

## 弱化拨打电话

针对打电话，实际上可以弱化一下，比如仅仅换气拨号界面，自动填充好手机号，由用户决定是否拨号，而不是直接把电话打出去，这样可以避免申请PHONE_CALL的权限.

```
Intent intent = new Intent(Intent.ACTION_DIAL);
intent.setData(Uri.parse("tel:"+"13717830629"));
startActivity(intent);
```
