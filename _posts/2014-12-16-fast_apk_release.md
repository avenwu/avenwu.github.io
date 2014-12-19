---
layout: post
title: "极速打包【shell版】"
description: "android快速批量打包"
category: 
tags: [android]
---
{% include JB/setup %}

###前言
前阵子无意间看到美团的技术文章，一口气读了几篇java、android相关的博文，写的都非常不错，其中有一篇讲得是android渠道包的问题，抱着好奇心读完全文，文中提到了几种渠道包生成方式，从ant+for循环，maven，gradle, zip+python,随着时间的迁移，不断在优化打包方式以满足项目需求，结合个人经历也确实如此。  

本文接着zip+python方式打包的思路介绍一下gradle+zip+shell的打包，一来笔者不懂python，而来在之前已经写过结合shell脚本的两种android打包，因此对shell更感兴趣。

###思路
这里所说的zip+python是笔者自己取得简称，因为涉及到了对zip压缩的处理，同时用的是python中的zip接口实现.  
同理gradle+zip+shell也就是利用shell命令处理zip包，以此得到我们期待的众多apk，apk本质就是zip压缩包，所以可以直接用标准zip处理。  
此法的基石在于：
	
	目前在apk中的META-INF目录中增加一个无内容的空文件后apk无需重新签名也可以正常安装运行

当然以后这种取巧的方式能不能行的通要看之后的android编译工具。

* 第一步通过gradle生成一个标准的release apk，其他方式也可。
* 有了标准的apk包之后我们，拷贝一份，并在其中添加一个表示渠道名称的文件，为了识别方便，可以定义自己的文件名规则，本文采用xxx.channel的格式，其中xxx为渠道名称，.channel为后缀。这样就得到了一个新的渠道包。
* 修改项目代码，动态获取该文件，从而得到渠道名


###开工
下面可以开始码代码了，脚本基于windows+cygwin编写，mac下测试无误。（下载cygwin时注意需要额外勾选zip和unzip两个工具）  
完整脚本请参考[buildtool.git](https://github.com/avenwu/buildtool.git)，此处经提取关键部分  

1. 通过gradle构建的标准项目，文通过如下命令来生成apk，目标apk默认位于app/build/outputs/apk/xxx  
	
	./gradlew clean assembleRelease

2. 创建存放渠道包的目录，简单起见就叫做apk，同时拷贝已经生成的apk备用，在cygwin中经常会出现文件的权错误，因此，我们需要在添加读写权限给apk

	rm -rf apk
    mkdir apk  
    cp build/outputs/apk/*_*.apk apk/pregnancy.apk  
    cd apk  
    chmod a+rw pregnancy.apk

3. 生成一个新的渠道包，根据当前的渠道名添加文件至apk压缩包内，这里用到了zip、unzip，相关用法不熟悉的google补一下吧

	cp app.apk pregnancy_$1.apk && cp META-INF/pregnancy.channel META-INF/$1.channel  
    zip -r pregnancy_$1.apk META-INF/$1.channel  
    rm META-INF/$1.channel
4. 仔细的你可能会问渠道名在哪里获取，$1什么意思？    
这里我们从预定义好的channels.properties中读取market_channel的值，并且以“,”分割得到渠道名的数组，而updateApk是我们定义的方法，方法体的内容即第三步中的代码。

	channels=(${market_channels//,/ })  
    for channel in ${channels[@]};do  
       updateApk $channel &  
    done

###优化
这里其实还有两个地方可以优化

1. 多核电脑可以考虑继承多线程，文中用的是background job和多线程应该还是有区别的，最好能控制线程数，已达到最优效果。
2. ant替换gradle可以减少第一个包生成的时间，目前默认情况下gradle编译要比ant慢一些。

###结语
相比较来说这种取巧的方式得到apk的速度非常快，相同配置的情况下生成越多包优势越明显，瓶颈只在于磁盘读取速率和内存。  
经实测，90个包在windows+cygwin+8g内存+ssd磁盘配置下，耗时约4分半，生成第一个包的时间约为3分钟左右，虽然没有所谓的1分钟900个包那么神奇，但是已经甩开常规循环打包几条大街了。
