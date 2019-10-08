---
layout: post
title: "解答android.jar问题引发的思考"
description: "android.jar代码的打包情况分析"
header_image: /assets/img/2019-10-08-01.jpg
keywords: ""
tags: []
---
{% include JB/setup %}
![img](/assets/img/2019-10-08-01.jpg)
* 目录
{:toc #markdown-toc}

## 背景
最近被一个小伙伴问了奇怪的一个jar的问题：
> 我有一个疑问，apk里面会包括系统的类，比如Activity.class。 
但是android.jar 并不是所有的类，都会打入进去apk；

![](/assets/images/android-jar-question.jpg)

**为什么说奇怪的呢？**
因为笔者一开始认为这是一个常识无需解释，但是小伙伴除了问题之外还附带了一个"证据"佐证。

![](/assets/images/524BD1B359FC05844B2C3045C70A7E8E.jpg)


## android.jar与打包
这里提的android.jar其实是SDK自带的Framework代码，类似JDK提供的rt.jar
，一般来说Framework的代码是不会随app打包的，它属于系统的代码，全局共享。

但是面对"证据"，我们很容易相信真的有sdk的代码被打包到apk当中，因此在"简单"回答了问题之后，笔者也忍不住分析了一把”证据“

> 1. 没有官方文档（估计以后也不会提供这种文档）
2. 你说的android.jar的打包问题，其实是想问能不能替换实现类吧
3. 没有实际观察过，个人理解：系统服务API都不会打进去，app业务相关的，比如四大组件之类都会进去

*注意，我这个”简单“回答是不太正确的。*

## 验证
首先当然是找一个apk亲自看一下他的内部代码，可以通过JD-GUI之类的反编译工具，也可以直接使用dexdump查看代码情况：
![](/assets/images/reverse-apk-andoird-jar.png)

可以看到反编译的结果中根本不存在android.app包名下的代码，只有扩展包androidx和v4。

进一步通过dexdump分析,同样不存在：
```
aven-mac-pro-2:.apk aven$ cat dump-classes.dex.txt |grep "Class descriptor"|grep 'Landroid/'
  Class descriptor  : 'Landroid/support/v4/os/ResultReceiver$1;'
  Class descriptor  : 'Landroid/support/v4/os/ResultReceiver$MyRunnable;'
  Class descriptor  : 'Landroid/support/v4/os/ResultReceiver;'
  Class descriptor  : 'Landroid/support/v4/graphics/drawable/IconCompatParcelizer;'
  Class descriptor  : 'Landroid/support/v4/os/ResultReceiver$MyResultReceiver;'
aven-mac-pro-2:.apk aven$ cat dump-classes.dex.txt |grep "Class descriptor"|grep 'Landroid/app/Activity;'
aven-mac-pro-2:.apk aven$ 

```

可以得出一个结论，apk确实不包含android.jar的代码，那么”证据“截图是怎么回事？

## 假象解释
从截图看，可以知道使用的是AndroidStudio自带的APK Analyzer，这个工具具备部分逆向能力，特别是对分析arsc，xml预览什么的，当然还有方法数，文件大小的统计。
截图中展示的android.app相关代码截图实际上就是Framework内的代码，在这里被显示的原因是：

> APK Analyzer展示了方法定义和引用，但是被展示出来，并不代表类被定义。一个类的定义/实现，如果不存在与apk中，那么就不能说是这个类被打进了apk中。所以通过APK Analyzer并不能证明一个类是否存在于apk中，只能确认是否又被引用到。

## 小结
看似很小的问题，如果不好好理解基础的话，也容易产生困扰。顺便附带了一篇SO，里面介绍了android.jar的其他一些信息。

* [what-is-android-jar-what-does-it-include](https://stackoverflow.com/questions/50283073/what-is-android-jar-what-does-it-include)