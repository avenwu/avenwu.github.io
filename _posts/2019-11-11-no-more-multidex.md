---
layout: post
title: "这一次和dex分包说再见"
description: ""
header_image: /assets/img/2019-11-11-01.jpg
keywords: ""
tags: [andorid]
---
{% include JB/setup %}
![img](/assets/img/2019-11-11-01.jpg)

* 目录
{:toc #markdown-toc}

从dex爆炸，到Google的multidex 2014年横空出世，再到app的分包斗争，gralde的迭代。为了兼容Android 4.4以下的老设备，分包方案层出不穷。

## 常规方案
基于multidex分包技术，我们总是需要工维护一份keep list, 来决定哪些类是一定要放到主dex中的。 
维护这份清单有手工人肉和脚本自动化两种方案。

> 手工方案: 通过hook gradle task, 调整系统生成的mainDexList.txt

这个方案的弊端很明显。随着时间的推移和业务的迭代，可能冷不丁就会出现一次主dex超标，开发人员就会很被动，需要立刻解决问题，否则直接应用项目的持续编译。

为了减少出现这种情况，我们做过多次hook优化，核心思路都是希望通过技术方案，低成本的让启动流程尽可能少的关联类，这样编译工具便不会把大量的类打入主包：
1. 比如精简启动流程
2. 又比如引入反射，断开类的直接引用关系
3. 再比如引入剔除清单，在最后一步强制将某些类踢出mainDexList.txt；
4. xxx

在这个过程中，对gradle的hook需要保持版本对齐，比如collectComponent在3.x已经不存在。

> 个性化依赖分析：通过定义个性化的依赖统计算法，实现启动引用类的最小化

这个方案理论可行，可以借鉴sdk提供的标准统计脚本，做二次开发，但是这个成本也不小；并且和系统脱钩有可能埋入隐藏的坑。

> 进程挂起方案：这个方案最早是脸书运用，后续微信，头条等国内App也开始采用

这个方案最大的特点就是不语系统做对抗，完全借用标准的分包编译工具。通过业务执行流程让安装dex前依赖减少，从而实现配置简化。

## Dexing分包
[Dexing](https://github.com/hacktons/dexing)项目正是基于进程挂起的思路，实现的一套开源方案。

核心流程如下图所示

![](/assets/images/multidex.png)

简单介绍一下：
* 当首次安装app时，低版本系统需要执行multidex才能保证类都找得到
* 这有两个问：首先要能正从打出包；其次安装很耗时
* `dexing`解决打出包是通过开启一个独立进程执行multidex，这个进程在逻辑上保证了没有其他非dex安装的业务，因此依赖集合非常稳定，并且很小
* `dexing`解决耗时问题，时间上没有本质解决，但是通过引入加载页面保证了安装过程不会黑屏/白屏，从而提高了一点点的用户体验

在使用层面，`Dexing`提供了与`Mutildex`一致的API,因此只需要将原来的安装过程，重新导包即可，
```java
import cn.hacktons.dexing.Multidex;

public class CustomApplication extends Application {
    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        Multidex.install(this);
    }
}
```

## 小结
很多时候我们都利用hook来规避问题，但有时候会出现尴尬的场面:
> hook一时爽，填坑火葬场😂

最后如果你阅读到了这篇文章，欢迎使用[Dexing](https://github.com/hacktons/dexing)。
