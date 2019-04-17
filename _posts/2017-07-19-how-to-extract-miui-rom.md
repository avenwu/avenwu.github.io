---
layout: post
title: "【MIUI】从零开始，ROM拆包实践"
description: "手动解析MIUI ROM，并提取相关资源"
header_image: /assets/img/2017-07-19-01.png
keywords: "ROM，解压"
tags: [ROM]
---
{% include JB/setup %}
![img](/assets/img/2017-07-19-01.png)

# 前言

近期Nexus 5下岗，更换了一部小米手机，经过一段时间使用，整体感觉不错，在使用其自带的音乐播放器时，发现有一个波谱挺有意思，因此有了本文，剖析MIUI ROM的介绍；

# 准备工作

既然要拆包，还是拆ROM，那么肯定会用到一些工具和资源；

* 下载MIUI ROM
* macOS/PC
* sdat2img

# MIUI ROM

下载一个你需要的分析的ROM包，笔者手机上安装的是MIUI 8.5,所以直接去找到他的对应ROM

* ROM 下载地址：[V8.5.3.0.MAGCNED ](http://bigota.d.miui.com/V8.5.3.0.MAGCNED/miui_MI5S_V8.5.3.0.MAGCNED_bd76d65057_6.0.zip)
* 如果链接失效了，可以去论坛重新找一下：[MIUI 小米手机5s](http://www.miui.com/download-319.html)

得到的ROM包大概长这样：

{% highlight bash %}
Archive:  miui_MI5SPlus_V8.5.3.0.MBGCNED_e77b4138cb_6.0.zip
signed by SignApk
  Length     Date   Time    Name
 --------    ----   ----    ----
        0  01-01-09 00:00   system.patch.dat
      127  01-01-09 00:00   META-INF/com/android/metadata
   624056  01-01-09 00:00   META-INF/com/google/android/update-binary
     5967  01-01-09 00:00   META-INF/com/google/android/updater-script
   342576  01-01-09 00:00   META-INF/com/miui/miui_update
 23524682  01-01-09 00:00   boot.img
  8965560  01-01-09 00:00   cust/app/customized/ota-miui-MiGalleryLockscreen/ota-miui-MiGalleryLockscreen.apk
 17607096  01-01-09 00:00   cust/app/customized/ota-miui-XiaomiSmartHome/ota-miui-XiaomiSmartHome.apk
 12530974  01-01-09 00:00   cust/app/customized/partner-BaiduSpeechService/partner-BaiduSpeechService.apk
 29056060  01-01-09 00:00   cust/app/customized/partner-MiShop/partner-MiShop.apk
 14120990  01-01-09 00:00   cust/app/customized/partner-XMRemoteController/partner-XMRemoteController.apk
 15971598  01-01-09 00:00   cust/app/customized/partner-Zixun/partner-Zixun.apk
      928  01-01-09 00:00   cust/app/vanward_applist
      105  01-01-09 00:00   cust/cust/cn/cust.prop
      521  01-01-09 00:00   cust/cust/cn/ota_customized_applist
      135  01-01-09 00:00   cust/cust/cn/ota_customized_channellist
      281  01-01-09 00:00   cust/cust/cn/ota_recommended_applist
      113  01-01-09 00:00   cust/cust/cn_chinamobile-cta/cust.prop
      239  01-01-09 00:00   cust/cust/cn_chinamobile-cta/ota_customized_applist
      252  01-01-09 00:00   cust/cust/cn_chinamobile-cta/ota_recommended_applist
      107  01-01-09 00:00   cust/cust/cn_chinamobile/cust.prop
      239  01-01-09 00:00   cust/cust/cn_chinamobile/ota_customized_applist
      252  01-01-09 00:00   cust/cust/cn_chinamobile/ota_recommended_applist
      167  01-01-09 00:00   cust/cust/cn_chinaunicom/cust.prop
      521  01-01-09 00:00   cust/cust/cn_chinaunicom/ota_customized_applist
      135  01-01-09 00:00   cust/cust/cn_chinaunicom/ota_customized_channellist
      281  01-01-09 00:00   cust/cust/cn_chinaunicom/ota_recommended_applist
       60  01-01-09 00:00   cust/cust/cn_cta/cust.prop
      521  01-01-09 00:00   cust/cust/cn_cta/ota_customized_applist
      135  01-01-09 00:00   cust/cust/cn_cta/ota_customized_channellist
      281  01-01-09 00:00   cust/cust/cn_cta/ota_recommended_applist
    57198  01-01-09 00:00   file_contexts
   421888  01-01-09 00:00   firmware-update/BTFM.bin
101527552  01-01-09 00:00   firmware-update/NON-HLOS.bin
 16777216  01-01-09 00:00   firmware-update/adspso.bin
   205496  01-01-09 00:00   firmware-update/cmnlib.mbn
   260392  01-01-09 00:00   firmware-update/cmnlib64.mbn
    51264  01-01-09 00:00   firmware-update/devcfg.mbn
  1987664  01-01-09 00:00   firmware-update/emmc_appsboot.mbn
   263672  01-01-09 00:00   firmware-update/hyp.mbn
   357264  01-01-09 00:00   firmware-update/keymaster.mbn
    57352  01-01-09 00:00   firmware-update/lksecapp.mbn
  1540484  01-01-09 00:00   firmware-update/logo.img
    42848  01-01-09 00:00   firmware-update/pmic.elf
   229420  01-01-09 00:00   firmware-update/rpm.mbn
   155555  01-01-09 00:00   firmware-update/splash.img
  1667072  01-01-09 00:00   firmware-update/tz.mbn
  1829416  01-01-09 00:00   firmware-update/xbl.elf
2568351744  01-01-09 00:00   system.new.dat
     1475  01-01-09 00:00   system.transfer.list
     1594  01-01-09 00:00   META-INF/com/android/otacert
     4594  01-01-09 00:00   META-INF/MANIFEST.MF
     4647  01-01-09 00:00   META-INF/CERT.SF
     1634  01-01-09 00:00   META-INF/CERT.RSA
 --------                   -------
2818552400                   54 files
{% endhighlight %}

# 挂载ext4

可以看到东西还不少，主要内容如下

* boot.img
* cust
* firmware-update
* system.new.dat
* ststem.transfer.list

从命名来看，我们需要提取得东西应该是在system.new.dat里面；  
我们看下这个是什么文件：  

{% highlight bash %}
system.new.dat: Linux rev 1.0 ext4 filesystem data (extents) (large files)
{% endhighlight %}

是个Linux下的ext4文件；如此我们可以尝试用mount挂载这个文件，

{% highlight bash %}
mount -t ext4 -o loop system.new.dat mount-img/
{% endhighlight %}

不过我们的挂载失败了：

{% highlight bash %}
mount: exec /Library/Filesystems/ext4.fs/Contents/Resources/mount_ext4 for /Users/aven/Downloads/miui_MI5SPlus_V8.5.3.0.MBGCNED_e77b4138cb_6.0/mount-img: No such file or directory
{% endhighlight %}

经过漫长的Google，我们发现在macOS下是不能直接挂载ext4文件的，需要借助osx fuse来完成；

# OSX fuse

为了挂载ext4的镜像文件，需要安装fuse和ext4fuse;

* fuse的安装可以通过installer，[OSX fuse](https://osxfuse.github.io/)
* ext4fuse的安装可以通过homebrew，`brew install ext4fuse`

安装完之后，挂载就通过ext4fuse执行：

{% highlight bash %}
aven-mac-pro-2:miui_MI5SPlus_V8.5.3.0.MBGCNED_e77b4138cb_6.0 aven$ ext4fuse system.new.dat mount-img/
aven-mac-pro-2:miui_MI5SPlus_V8.5.3.0.MBGCNED_e77b4138cb_6.0 aven$ ls mount-img/
ls: : Device not configured
{% endhighlight %}

虽然我们挂载了文件，并且没有提示错误，但是查看挂载后的目录发现并没有生效；

# system.new.dat

这件事情告诉我们，不了解ROM的内容而直接尝试挂载内容有点难度；最后在xda轮胎找到了一些线索：
system.new.dat文件虽然显示的是ext4文件，但是不能直接使用，我们需要通过sdat2img将相关文件转换一下，得到最终的可以挂载的文件；

这个资料网上搜一下发现有很多，中文英文都有，不过基本都是类似的，也没有找到比较权的官方说明文档；

{% highlight bash %}

aven-mac-pro-2:miui_MI5SPlus_V8.5.3.0.MBGCNED_e77b4138cb_6.0 aven$ chmod +x sdat2img.py 
aven-mac-pro-2:miui_MI5SPlus_V8.5.3.0.MBGCNED_e77b4138cb_6.0 aven$ ./sdat2img.py system.transfer.list system.new.dat system.img
sdat2img binary - version: 1.0

Android Marshmallow 6.x detected!

Copying 32770 blocks into position 0...
Copying 2 blocks into position 32961...
Copying 32064 blocks into position 33471...
Copying 2 blocks into position 65536...
Copying 32257 blocks into position 66046...
Copying 2 blocks into position 98304...
Copying 2 blocks into position 98497...
Copying 32064 blocks into position 99007...
Copying 2 blocks into position 131072...
Copying 32257 blocks into position 131582...
Copying 2 blocks into position 163840...
Copying 2 blocks into position 164033...
Copying 32064 blocks into position 164543...
Copying 2 blocks into position 196608...
Copying 32257 blocks into position 197118...
Copying 2 blocks into position 229376...
Copying 2 blocks into position 229569...
Copying 32064 blocks into position 230079...
Copying 2 blocks into position 262144...
Copying 32257 blocks into position 262654...
Copying 2 blocks into position 294912...
Copying 2 blocks into position 295105...
Copying 32064 blocks into position 295615...
Copying 2 blocks into position 327680...
Copying 32257 blocks into position 328190...
Copying 2 blocks into position 360448...
Copying 32257 blocks into position 360958...
Copying 2 blocks into position 393216...
Copying 32257 blocks into position 393726...
Copying 2 blocks into position 425984...
Copying 32257 blocks into position 426494...
Copying 2 blocks into position 458752...
Copying 32257 blocks into position 459262...
Copying 2 blocks into position 491520...
Copying 32257 blocks into position 492030...
Copying 2 blocks into position 524288...
Copying 32257 blocks into position 524798...
Copying 2 blocks into position 557056...
Copying 32257 blocks into position 557566...
Copying 2 blocks into position 589824...
Copying 14602 blocks into position 590334...
Copying 2 blocks into position 622592...
Copying 2 blocks into position 655360...
Copying 2 blocks into position 688128...
Copying 2 blocks into position 720896...
Copying 2 blocks into position 753664...
Copying 26056 blocks into position 754174...
Copying 6153 blocks into position 780231...
Skipping command zero...
Skipping command erase...
Done! Output image: /Users/aven/Downloads/miui_MI5SPlus_V8.5.3.0.MBGCNED_e77b4138cb_6.0/system.img
{% endhighlight %}

转换完格式后，再次挂载文件，成功了：

{% highlight bash %}
aven-mac-pro-2:miui_MI5SPlus_V8.5.3.0.MBGCNED_e77b4138cb_6.0 aven$ ext4fuse system.img mount-img/
{% endhighlight %}

查看挂载后的目录中，发现了我们想要的文件

{% highlight bash %}
total 632
dr-xr-xr-x@  19 root  wheel    4096 Jan  1  1970 .
drwx------   15 aven  staff     510 Jul 19 11:28 ..
dr-xr-xr-x  123 root  wheel    4096 Jan  1  2009 app
dr-xr-xr-x    2 root  2000     8192 Jan  1  2009 bin
-r--r--r--    1 root  wheel   10695 Jan  1  2009 build.prop
dr-xr-xr-x    9 root  wheel    4096 Jan  1  2009 data-app
dr-xr-xr-x   30 root  wheel    4096 Jan  1  2009 etc
dr-xr-xr-x    2 root  wheel    8192 Jan  1  2009 fonts
dr-xr-xr-x    6 root  wheel    4096 Jan  1  2009 framework
dr-xr-xr-x    7 root  wheel   12288 Jan  1  2009 lib
dr-xr-xr-x    6 root  wheel   12288 Jan  1  2009 lib64
dr-x------    2 root  wheel    4096 Jan  1  1970 lost+found
dr-xr-xr-x    6 root  wheel    4096 Jan  1  2009 media
dr-xr-xr-x   72 root  wheel    4096 Jan  1  2009 priv-app
-r--r--r--    1 root  wheel  211711 Jan  1  2009 recovery-from-boot.p
dr-xr-xr-x    5 root  wheel    4096 Jan  1  2009 rfs
dr-xr-xr-x    3 root  wheel    4096 Jan  1  2009 spaces
dr-xr-xr-x    3 root  wheel    4096 Jan  1  2009 tts
dr-xr-xr-x    8 root  wheel    4096 Jan  1  2009 usr
dr-xr-xr-x    8 root  2000     4096 Jan  1  2009 vendor
dr-xr-xr-x    2 root  2000     4096 Jan  1  2009 xbin
{% endhighlight %}

# 小结

接下来的事情就很明确了，在app目录下就是我们的系统/预装软件,包括音乐播放器，这个我们留到下一篇在分析：）

# 参考

* [Extract system.new.dat of Marshmallow and Lollipop (easily)](https://forum.xda-developers.com/android/help/extract-dat-marshmallow-lollipop-easily-t3334117)
* [sdat2img](https://github.com/xpirt/sdat2img)
* [Android5.0的更新包中system.new.dat文件的解包](http://blog.csdn.net/howellzhu/article/details/41967523)
* [安卓教程 Android 5.0(Lollipop)system.new.dat解包工具及方法](http://htcui.com/4505.html)
