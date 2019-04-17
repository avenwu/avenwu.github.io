---
layout: post
title: "android流量统计"
header_image: /assets/img/2016-03-06-01.jpg
keywords: "流量, 统计"
description: "android traffic data"
tags: [android]
---
{% include JB/setup %}

![img](/assets/img/2016-03-06-01.jpg)

## 前言

用过各种手机清理软件的android用户都知道，这些软件往往都可以查询应用的流量使用情况；  
从开发者的角度来说，第一反应很可能是“数据从哪里来的？怎么算的？”，本文就来分析一下如何获取Android设备上流量使用情况。

## 分析
安装app后随便浏览网页，消耗点流量，打开检测软件，看到类似这样的统计数据。

![Screenshot_20160223-175015.png]({{ site.baseurl }}/assets/images/Screenshot_20160223-175015.png)

先去开发者网站看看有没有相关接口
[http://developer.android.com/reference/android/net/TrafficStats.html](http://developer.android.com/reference/android/net/TrafficStats.html)

Android提供了TrafficStats接口，可以获取一些简单的数据，注意这些数据是累计的：

* getTotalRxBytes()
* getTotalTxBytes()
* getMobileTxBytes()
* getMobileRxBytes()
* getUidTxBytes(int uid)

方法名看起来很奇怪，基本上是tx，rx配套存在，tx代表的是上行或者说是上传（t=>transmitted），rx代表的是下行(r=>received);

因此通过系统的接口我们可以知道当前系统/app累计消耗的数据包和上传的数据包，移动网络下的消耗情况；

如果要生成条形图，那么需要定期吧消耗的数据存储起来，这样随着时间推移，就可以绘制出每天的流量消耗；

根据目前信息可以很方便的计算出自身app的数据使用情况，那其他app的数据呢？当然也是可以的通过制定uid，每个应用在运行过程中会有一个uid，这个uid标示了当前进程；

要获取其他app的uid好像不是太容易，现在去看看其他应用是怎么获取到的；

## 统计类app拆包分析
本文分析的onavo count，拆完包发现onavo是个好同志啊，为什么呢？应为混淆做的很好啊，基本都是明文，直接可读。

![package_folder.jpg]({{ site.baseurl }}/assets/images/package_folder.jpg)

通过关键查找，最后在这里发现了有价值的东西

![read_xt_qtaguid.jpg]({{ site.baseurl }}/assets/images/read_xt_qtaguid.jpg)

和很多黑技术一样，这里也可以直接读取系统文件，如果对linux体系熟悉的话应该知道文件系统内各个目录的作用，当然笔者并不熟悉linux内核；

连接设备，adb看一下这个文件里面到底是什么

![xt_qtaguid_content.jpg]({{ site.baseurl }}/assets/images/xt_qtaguid_content.jpg)

输出了很多数据，每一行通过空格分割，基本上是个表格的意思，第一行相当于表头，表示每个纵列的含义，第二行开始表示每个每个进程的数据情况；

那到底哪些列使我们关注的呢？其实就是关键的rx，tx，我们看看onavo用了哪些数据

![read_stats_file.jpg]({{ site.baseurl }}/assets/images/read_stats_file.jpg)

不得不感概，作者是好人啊，大写的赞👍

现在其实就可以从这个表中获取每个应用的数据使用情况了，但是我们还是不知道特定应用的uid啊，别急只要知道package，可以用dumpsys命令把它过滤出来

	dumpsys package com.example.myapp | grep userId=

得到uid后，可以过滤第一个表

	cat /proc/net/xt_qtaguid/stats | grep 10060

最后就是将过滤出来的数据求和；

下面以UC浏览器做演示：

1|root@vbox86p:/ # dumpsys package com.UCMobile.intl.x86 | grep userId=        
    userId=10068 gids=[3003, 1028, 1015]

得到uid为10068

```
root@vbox86p:/ # cat /proc/net/xt_qtaguid/stats | grep 10068                   
16 eth1 0x0 10068 0 2726 27 3028 60 2726 27 0 0 0 0 3028 60 0 0 0 0
17 eth1 0x0 10068 1 41345707 40808 1047668 19955 41345707 40808 0 0 0 0 1047668 19955 0 0 0 0
32 lo 0x0 10068 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
33 lo 0x0 10068 1 120 3 180 3 120 3 0 0 0 0 180 3 0 0 0 0
```

根据该uid得到了四行数据；

表头是

```
idx iface acct_tag_hex uid_tag_int cnt_set rx_bytes rx_packets tx_bytes tx_packets rx_tcp_bytes rx_tcp_packets rx_udp_bytes rx_udp_packets rx_other_bytes rx_other_packets tx_tcp_bytes tx_tcp_packets tx_udp_bytes tx_udp_packets tx_other_bytes tx_other_packets
```

可以看到其中的rx_bytes位于第5位, tx_bytes位于第7位

## 结束语

当然，如果继续对onavo脱裤，应该能挖到其他的好玩东西；另外过过google，找到了两篇和linux内这个统计文件相关的文章

* http://stackoverflow.com/questions/12904809/tracking-an-applications-network-statistics-netstats-using-adb
* https://testerhome.com/topics/2068

