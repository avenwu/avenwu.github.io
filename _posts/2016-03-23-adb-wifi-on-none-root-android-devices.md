---
layout: post
title: "ADB无线调试-免root"
header_image: /assets/img/2016-03-23-01.jpg
keywords: "adb wifi 无需root ADB WIFI on none root android devices"
description: "adb重定向，免root远程调试[ADB WIFI on none root android devices]"
category: 
tags: [android]
---
{% include JB/setup %}
![img](/assets/img/2016-03-23-01.jpg)

## 前言
平时开发的时候经常使用ADB WIFI，在局域网内可以实现ADB无线调试，但是有一个问题，过去一直是在root后的手机上使用，换手机后不想root，怎么办？

## 新设备福音【不保证所有设备可行性】
世界很奇妙，以前试了很多方法都无法通过简单操作解决非root手机的调试，那些市场上的所谓无需root的ADB WIFI多数没什么软用；
今天再次Google了一把，找到了解决方法，利用的任然是adb重定向相关；

下面已小米PAD为例：

* 通过USB链接设备与电脑，确保正确链接后，adb devices可以看见你的设备；

{% highlight shell %}
avens-MacBook-Pro:~ aven$ adb devices
List of devices attached
24316448	device
{% endhighlight %}

* 查询该设备的IP地址, 注意这里用不了ifconfig，需要用ip命令，不信你试试；

{% highlight shell %}
avens-MacBook-Pro:~ aven$ adb shell
shell@mocha:/ $ ifconfig
255|shell@mocha:/ $ ip -f inet addr show wlan0
6: wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    inet 10.0.101.192/16 brd 10.0.255.255 scope global wlan0
       valid_lft forever preferred_lft forever
{% endhighlight %}

* adb重定向，adb tcpip 5555

* 现在可以断开USB连接了，用上你熟悉的adb connect
{% highlight shell %}
adb connect 10.0.101.192
{% endhighlight %}

没有意外的话应该就连接上了，如果提示操作超时，那多试几次，还不行的话，自谋多福了；

* 查看连接的设备
{% highlight shell %}
shell@mocha:/ $ avens-MacBook-Pro:~ aven$ adb devices
List of devices attached
10.0.101.192:5555	device
{% endhighlight %}

好了现在可以抛开数据线，愉快的调试起来吧！！

## 小结
adb有很多命令，感兴趣可以去开发者官网查阅
最后附上SO上，哥们的原回复:

The steps are correct, with one small part different: the connect step has to be done after taking out the cable. To reiterate follow the steps exactly as below and it will work for non-rooted devices also. I tested it with several non-rooted devices including Moto G, Nexus 1, Videocon etc.

Attach mobile via USB and type:

adb tcpip 5555
To find the mobile ip type:

adb shell ip -f inet addr show wlan0
The ip address will be shown in second line like this:

inet 192.168.1.233/24 brd 192.168.1.255 scope global wlan0
where 192.168.1.233 is the ip address of your mobile.

Remove USB cable and type:

adb connect mobile-ip:5555


## 参考
* [http://stackoverflow.com/questions/14358882/connecting-adb-using-wifi-for-non-rooted-device](http://stackoverflow.com/questions/14358882/connecting-adb-using-wifi-for-non-rooted-device)




