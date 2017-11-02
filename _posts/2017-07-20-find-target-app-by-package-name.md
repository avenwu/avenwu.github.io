---
layout: post
title: "【MIUI】从ROM提取音乐播放器"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2017-07-21-01.jpg
keywords: "小米音乐播放器"
tags: [ROM]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2017-07-21-01.jpg)

# 前言

接上文 [【MIUI】从零开始，ROM拆包实践]({{ site.baseurl }}/2017/07/19/how-to-extract-miui-rom),现在我们已经成功把ROM挂载，并得到系统的预装程序和其他服务；  
本文将顺一步，开始定位我们的目标程序“音乐播放器”。

# 资源预览

我再看一下，ROM里面的这些目录：

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

内容这么多，如何定位播放器apk的归档位置，这是一个效率问题？  
可以看到app/下面有很多程序，如果一个个去验证的话将非常痛苦，所以我们肯定要用脚本来做这些重复性的工作；

# 检测脚本

我们知道小米播放器的包名是：`com.miui.player`, 所以思路很简单，检测ROM中所有apk，输出包名和预期相符的apk的归档位置；  
大致的流程如下：

![miui-rom-check-apk]({{ site.baseurl }}/assets/images/miui-rom-check-apk.png)

检查一个apk的包名比较简单，之前也已经提到过，[快速解析apk的版本信息]({{ site.baseurl }}/2016/12/08/aapt-tips/)

```
aapt dump badging $file |grep $package
```

循环遍历文档的命令很多，这里我们需要输出命中的目标所在的apk位置，所以直接对指定目录进行了枚举；如此只需要把挂载后的目录作为入参，就可以递归检测期下所有的apk，当然在实际操作中，扫描一个ROM的所有apk在这里并不是很快，不过还是可以接受，比起手工查找方便不少；

{% highlight bash %}
# search recurrsively
function search(){
	keywords=$1
	file=$2
	if [ -d $file ]; then
		files=($file/*)
		for f in "${files[@]}"; do
			search $keywords $f
		done
	elif [ -f $file ]; then
		isApkHit $keywords $file
	fi
}
{% endhighlight %}

完整的脚本已经托管出来，可以前往下载：[updateGitUserRecurring](https://github.com/avenwu/tips/blob/master/updateGitUserRecurring)。

{% highlight bash %}
aven-mac-pro-2:Music aven$ searchAppPackage com.miui.player ~/Downloads/miui_MI5SPlus_V8.5.3.0.MBGCNED_e77b4138cb_6.0/mount-img/
/Users/aven/Downloads/miui_MI5SPlus_V8.5.3.0.MBGCNED_e77b4138cb_6.0/mount-img/ is set as searching folder
W/ResourceType(37630): No known package when getting value for resource number 0x01080093
ERROR getting 'android:icon' attribute: attribute is not a string value
W/ResourceType(37702): No known package when getting value for resource number 0x100c0007
ERROR getting 'android:label' attribute: attribute is not a string value
W/ResourceType(37876): Failure getting entry for 0x7f0d0204 (t=12 e=516) in package 0 (error -84)
ERROR getting 'android:label' attribute: attribute is not a string value
ERROR getting 'android:icon' attribute: attribute is not a string value
ERROR: dump failed because no AndroidManifest.xml found
W/ResourceType(38008): No known package when getting value for resource number 0x0104046e
ERROR getting 'android:label' attribute: attribute is not a string value
ERROR getting 'android:targetSdkVersion' attribute: attribute is not a string value
Found com.miui.player in /Users/aven/Downloads/miui_MI5SPlus_V8.5.3.0.MBGCNED_e77b4138cb_6.0/mount-img//priv-app/Music/Music.apk
package: name='com.miui.player' versionCode='93' versionName='2.9.1000' uses-permission:'com.miui.player.permission.MIPUSH_RECEIVE' launchable-activity: name='com.miui.player.ui.MusicBrowserActivity' label='' icon=''
W/ResourceType(38185): No known package when getting value for resource number 0x01040014
ERROR getting 'android:label' attribute: attribute is not a string value
search done
{% endhighlight %}

从输出来看，有一些错误信息，不过没关系，先忽略错误信息，可以看到播放APK的位置`miui_MI5SPlus_V8.5.3.0.MBGCNED_e77b4138cb_6.0/mount-img//priv-app/Music/Music.apk`

并不是在app目录中，而是pri-app目录。

# APK真身在哪里

现在打开Music.pak看下内容，资源文件都在，但是没有dex，原因是rom发出的时候对apk做了预处理，把dex优化为了odex，放在了oat的专门目录下，因此，想要分析dex内容，我们还需要odex转换会dex；这个就更麻烦一些，查了些资料，可以使用smali/baksmali工具做转换；

```
baksmali-2.2.1.jar
smali-2.2.1.jar
```
这两个是开源项目，并且一直在更新，真的很赞，同时这个工具在2.x之后做了重大更新，所以网上很多的使用说明都是不对的，可以直接参照github上的使用说明，结合help参数查看具体操作；

最后需要执行的操作可以简单总结如下，会得到目标dex，名为classes.dex：

```
# decompile odex to smail according to boot.oat an other framework dependency
java -jar $BAKSMALI x -b $BOOT -d $FRAMEWORK $ODEX

# compile the smali file into normal dex file, so we can use jd-gui tool 
java -jar $SMALI a out -o classes.dex
```

好了，摆好姿势，请开始你的表演：）

![smilek]({{ site.baseurl }}/assets/images/emoji-smile.png)

# 参考

* [https://github.com/JesusFreke/smali/wiki/SmaliBaksmali2.2](https://github.com/JesusFreke/smali/wiki/SmaliBaksmali2.2)
* [https://bitbucket.org/JesusFreke/smali/downloads/](https://bitbucket.org/JesusFreke/smali/downloads/)

