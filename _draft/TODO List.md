# TODO List
记录一些好玩的，值得分析的东西

## 长按自动识别二维码
微信中长按图片，如果是二维码，过一会就会出一个选项识别二维码，比较实用；

## SDK分析
* Facebook登录sdk，sina登录sdk
* Fabric统计sdk，umeng统计

# 4G代理的实现
## Small
看微信架构演进中发现的一个东西, 一款插件化的开发库
* [https://github.com/wequick/Small/wiki/Android](https://github.com/wequick/Small/wiki/Android)

* 调高app优先级放大，利用前台service

		从统计上报看，提高后的效果极佳。

		原理：Android 的前台service机制。但该机制的缺陷是通知栏保留了图标。

		对于 API level < 18 ：调用startForeground(ID， new Notification())，发送空的Notification ，图标则不会显示。

		对于 API level >= 18：在需要提优先级的service A启动一个InnerService，两个服务同时startForeground，且绑定同样的 ID。Stop 掉InnerService ，这样通知栏图标即被移除。

		这方案实际利用了Android前台service的漏洞。微信在评估了国内不少app已经使用后，才进行了部署。其实目标是让大家站同一起跑线上，哪天google 把漏洞堵了，效果也是一样的。


## 渠道包问题
渠道包的技术方案有很多种，本文汇总豌豆荚在处理apk时用到了appt这个android原生的资源打包工具，值得深挖一下aapt的其他好用点

* http://geek.csdn.net/news/detail/76488

## GitHub开源的跨平台桌面应用开发框架electron
* [https://github.com/electron/electron-api-demos](https://github.com/electron/electron-api-demos)
* [http://electron.atom.io/#get-started](http://electron.atom.io/#get-started)

## Java NIO
* 介绍NIO的不可多得的教程[http://tutorials.jenkov.com/java-nio](http://tutorials.jenkov.com/java-nio)
* 翻译了部分，第11节好长的文章，每天一点，翻译了N久
## 好书收集
人一旦不读书，就容易变得懒且蠢，从几位前辈出收罗了基本好书工空闲时阅读
* 《How Google Works》
* 《深入理解计算机系统（第二版）》
* 《程序员自我修养-链接、装载与库》
* 《计算机程序的构造和解释（第二版）》


## weekly

* [https://medium.com/@qhutch/android-simple-and-fast-image-processing-with-renderscript-2fa8316273e1#.tj53trl91](https://medium.com/@qhutch/android-simple-and-fast-image-processing-with-renderscript-2fa8316273e1#.tj53trl91)
* 【Done】google内部的reactive programming开源项目[https://github.com/google/agera](https://github.com/google/agera)

## 博客园用户反馈2016/05/04
用户反馈一个crash并附带了日志，周末找时间看看能不能处理一下，很久没发版了

## RxJava不易理解的几个概念

* 【Done】初略过了一遍RxJava Essentials 中文翻译版，有些地方还不是很理解，需要在看看英文原版，
* 【Done】sampling采样，和debounce很像，但是sample在采样的时间段汇总如果没有数据发射，那么在这个时间段内就为空，这和debounce不同
* 【Done】timeout与debounce，debounce是防抖，过滤一些数据，每次都会保证能返回一个数据
* 【Done】join的时间窗口问题,join操作A,B两个Observable数据源时，将第二个数据源B的每一项和A中已发射的数据做排列，每次都得到一组新的值；

## RxJava使用与原理学习
* 【Done】线程调度Scheduler [http://reactivex.io/documentation/scheduler.html](http://reactivex.io/documentation/scheduler.html)

## Diy code，开源库信息
* 一堆资源摘要
[http://diycode.cc/topics/60](http://diycode.cc/topics/60)
* 3D效果的动画
[https://github.com/danielzeller/Depth-LIB-Android-](https://github.com/danielzeller/Depth-LIB-Android-)
* 类似FloatingActionButton的实现库
[https://github.com/futuresimple/android-floating-action-button](https://github.com/futuresimple/android-floating-action-button)

## Base64原理，实现
分析Base64的设计原理

## Apk解析(apk安装过程会解析manifest)
PackageUtil.getPackageUnfo()

## vector drawable 兼容包
google出品必属精品，gradle2.0据说开始以新的方式支持vector

[http://android-developers.blogspot.com/2016/02/android-support-library-232.html](http://android-developers.blogspot.com/2016/02/android-support-library-232.html)

## 下拉刷新分析 (80% done)
国人写的一个刷新库，可以分析分析，看和Chrise Beane的实现差异  
[https://github.com/liaohuqiu/android-Ultra-Pull-To-Refresh.git](https://github.com/liaohuqiu/android-Ultra-Pull-To-Refresh.git)

## 动画分析
涉及一些贝塞尔曲线的运用，实现水波纹，gitub也有另外的一个水波纹TextView    
[http://blog.csdn.net/harvic880925/article/details/50995587](http://blog.csdn.net/harvic880925/article/details/50995587)

