---
layout: post
title: "开源|Magpie平台工具链开发实践"
description: "Magpie Workflow是一个Flutter开发的工具流，用于实现独立Flutter模块的创建，开发，编译，打包，上传流程；旨在简化Flutter混合开发的复杂度，成为连接开发者与Flutter的桥梁"
header_image: http://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/8cf29d5f-cc90-4450-ba8c-1fa27f9961ccworkflow-arc.png
keywords: "Flutter平台工具链"
tags: [Flutter]
---
{% include JB/setup %}
![img](http://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/8cf29d5f-cc90-4450-ba8c-1fa27f9961ccworkflow-arc.png)

* 目录
{:toc #markdown-toc}

本文根据直播内容整理而来，首发于58技术公众号。

[https://mp.weixin.qq.com/s/vbGKxPJ3YkpSBHEkCQVXzQ](https://mp.weixin.qq.com/s/vbGKxPJ3YkpSBHEkCQVXzQ)

---

全文大纲如下：

* Flutter混编现状
* Magpie解决方案
* Flutter容器隔离
* 踩坑与规划

## 1 Flutter混编现状

在过去的2019年，Flutter技术持续火热。各大公司都在加码投入Flutter技术的落地，闲鱼、快手以及头条等也已开始进行工程方面的深度定制。可以预见在2020年，Flutter生态将持续发展、壮大。

在开始之前，我们先抛出几个问题：

![Slide3.png](http://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/85ad4241-7c38-434a-aa7b-081440439f2cSlide3.png)

图1 思考

> 1. 为什么要引入Flutter技术？
> 2. 如何将Flutter落地到业务？
> 3. 如何支撑Flutter并行研发？

针对Flutter的技术优势，大家可以从官方介绍得到解答。他所具备的性能体验，研发效率，多端一致性以及开源可控等特性，都吸引着广大开发者。

但是，Flutter并不是万金油；就目前来说，每个团队是否采用该技术栈，影响因子很多，我们需要结合自身的业务和诉求加以考虑。

### 1.1 Native与Flutter

通过Flutter技术，从零开始打造一款产品，并产出体验一致的多端应用，基本上我们只需要遵循Flutter官方的指引说明。这种情况下，相对来说交叉少，没有什么技术包袱需要考虑。

在混合开发层面则复杂得多，大多数的团队在使用Flutter时，走的都是混合开发的方向。即在现有的Android/iOS应用基础上，部分业务通过Flutter开发。这样可以最大限度的保障现有业务的稳定，也可以实现基础能力的复用。

他带来的问题也是明显的，混合开发带来工程结构组织，性能等系列问题，我们面临第一个技术选型：

> 采用哪种混合开发方式来实现业务的混合开发。

截止2019年末，Flutter开发社区中主要的有三个流派来进行混合开发：

![image.png](http://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/b54447b4-7c81-4158-bfb2-7877155fef12image.png)

图2 混合开发模式

- 官方推出的**Add to app**功能，正式支持的版本为**1.12+**以上
- **Flutter Boot**混合开发方案
- 其他私有的**自研方案**，由于未开源/受众少，不再展开

无论哪种方案，我们在确定技术方案时，需要考虑到现有业务，以及我们的工程模式和研发效率问题。

### 1.2 痛点与方向

在Flutter实践过程中，有一些痛点成为了我们的**拦路虎**。

![Slide5.png](http://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/2dd72bf7-b08a-4a7b-a869-05b1c3bc1cc4Slide5.png)

图3 痛点

当尝试原生混合Flutter的时候，你会发现：

> 编译链更复杂了

为了开发Flutter，需要Flutter编译链，Android编译链，还有iOS的Xcode编译链。

> 源码工程更复杂了

类似的，一个工程中三端代码的出现，让人难以**心静自然凉**。

> 不稳定

我们早期曾经尝试过**Add to app**，但是发现他并不完善，构建过程会导致文件被删除，比如.android文件。

这些问题在大型工程中显得更加膈应，我们急需解决他们，于是团队沉浸到了Flutter源码研究当中。

### 1.3 工程模式

从历史的角度来看，更优雅的方案应该做到工程化，端与端的隔离。我们回顾下原生开发的情况。

![Slide7.png](http://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/5d746de3-c1f1-463f-b637-ad3ccb1502f7Slide7.png)

图4 工程模式

**小型工程**：只需要团队集中火力到Native侧，周期性迭代；

**大型工程**：如果引入混合开发，比如RN/Hybrid，那么除了Native团队，还会有业务的RN团队/Hybrid团队，最后产出业务维度的bundle。

> 如果把Flutter也按照这种模式迭代，很多问题就迎刃而解了

![Slide8.png](http://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/152bd56d-24a1-432a-9ba8-4d9cdfbe2e10Slide8.png)

图5 Flutter工程模式

## 2 Magpie解决方案

在Flutter工程化过程中，我们自研了**Magpie解决方案**，整体上Magpie Workflow是一个Flutter开发的工具流，实现独立Flutter模块的创建，开发，编译，打包，上传流程；

我们的初衷在于简化Flutter混合开发的复杂度，成为连接开发者与Flutter的桥梁，因此称为Magpie Workflow。

> 小百科: 在英文中，“Magpie Bridge”代表“鹊桥”，也就是中国古老的爱情神话《牛郎织女》里面，由喜鹊们汇成的一座桥。

### 2.1 研发背景

结合前面我们提到的混合编译的现状。我们还有一些研发的考量。首先我们看一下Flutter的编译问题。

正如你知道的，Flutter有一套热重载机制，包括`Hot reload`和`Hot restart`，这两把利器让我们编写Dart时，效率倍增，相比Native开发形式，几乎是御风而行，Android的Instant Run也有类似体验，但是不在一个级别。

然而，如果你进行`Cold reload`,即触发整个工程的构建，这个时候是非常难忍受的，你的Native工程根本没有改变，但是他却需要完整的assemble构建出安装包，如果你的Native工程本来就很庞大那么这个体验会无限放大。

我们整理了一张图表，按粗粒度对照了**Native开发**，**混合开发**，以及**并行开发**的编译耗时。

![Slide9.png](http://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/a52026d2-b8e1-4b6b-b4f1-5f060e9abc2fSlide9.png)

图6 研发背景

可以看到，在Native与Flutter混合开发时，时间是需要在Native基础上叠加的。他的`Cold reload`时间会很长，很长。

> 我们期望的是实现一种flutter的并行开发方式，剥离Native，从而减少不必要的编译耗时。

另一个问题，是如何进行可视化的分离开发，你可能用过很多命令工具，但是你应该不会希望在开发Flutter过程中需要去记忆很多命令名字，参数之类的吧。当然不排除**极客开发者**，这类高效的开发人员可能对会大量命令爱不释手。但是从大概率上来说，人是视觉动物，我们会更习惯使用可视化的方法来操作。

> 如果一个GUI的工具，即简洁，又有颜，会不会更容易被开发者接受。

![Slide10.png](http://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/ca0f272e-a3dd-4fd3-8f35-95c2586a1cd1Slide10.png)

图7 可视化

### 2.2 技术架构

现在我们一起来看一下我们自研的**Magpie**解决方案。还记得前面我们介绍过的**Add to app**和**Flutter Boot**吗，我们做一下横向对比，从直观上了解一下差异点。

![Slide15.png](http://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/7c6f192e-fd16-48d3-ab1c-fdea1864968eSlide15.png)

图8 横向对比

我们在研发Magpie时，针对现有的混合开发方案的问题做了定向优化。在工程源码和编译环境上面做了多视角的隔离，是的Flutter业务开发者和Native平台开发者能够专注在各自的重心。

同时重点支持了并行开发，实现了Flutter业务和Native原生的并行开发。得益于隔离方案，我们实现了Cold reload的更快速度。同时由于我们采用了dart技术栈，使用**Magpie解决方案**，不需要安装额外的编译环境。



整个Magpie技术架构基于分层设计思想，包括三部分，分别是：

* 脚手架工具CLI
* 可视化的Workflow Service
* 用于App接入的Magpie SDK

![Slide19.png](http://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/679f6993-83b5-415e-9c8c-89d24ee3f087Slide19.png)

图9 Magpie架构图一


![workflowarc.png](http://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/8cf29d5f-cc90-4450-ba8c-1fa27f9961ccworkflow-arc.png)

图10 Magpie架构图二

## 3 Flutter容器隔离

这一节，我们从使用者的角度，来看看怎么使用**Magpie**。

### 3.1 Magpie上手

![Slide16.png](http://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/2b2e0a7e-b524-42ce-b98a-c7df3828d276Slide16.png)

图11 Magpie开发模式

上图涵盖了工程创建，编译，运行，发布等环节。

1. 通过我们提供的脚手架命令，进行模板工程创建
2. 通过start命令，启动可视化的Workflow页面
3. 在Workflow中操作，如编译，发布等

![Slide21.png](http://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/b4ae6347-07e9-45b1-90ee-bf3f409d3891Slide21.png)

图12 Magpie工作流程

脚手架包括两个核心职能：创建工程，启动Workflow。除此之外也包含其他一些命令，整体构成了cli的指令集。

![Slide24.png](http://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/12221545-3867-4dd2-833a-b6deb5f43f77Slide24.png)

图13 脚手架指令集

### 3.2 Workflow使用

我们将开发过程涉及的操作，设计为了界面操作。这个灵感来自于ReactNative开发社区的Expo。

当我们启动项目后，项目便与Workflow产生了关联，我们在前端页面中进行开发。

![Slide26.png](http://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/13eb07a6-14e1-451b-afbc-cb547aded61eSlide26.png)

图14 Workflow功能

* 可以找到需要连接使用的设备
* 以JIT的形式编译Flutter Module
* 将模块Attach到App中进行预览和调试
* 以AOT的形式编译Flutter Module
* 可选的将AOT产物上传到Maven
* 如果使用过程出现了问题，别慌，在日志中发现线索，或反馈给我们

![Slide29.png](http://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/b9253155-88e7-4f38-8d50-84e74d7381faSlide29.png)

图15 Workflow产物

## 4 后续规划

在研发Magpie的过程中，我们也遇到了一些问题。

> JS转义后运行时崩溃

在Debug模式下开发正常，但是release之后的js代码中由于class的引用方式差异，会出现类定义的不同，从而引起崩溃。

> Flutter组件体验偏移动端

目前Flutter提供的基础Widget如果运用到PC上，效果比较勉强，很多H5/CSS常见效果有缺失。比如文本复制能力，目录选择。

> Flutter Web生态

目前Flutter Web处于Beta状态，正如官方的建议，不建议使用到商业生产环境中。但是整体来说，可用性还是比较强的，我们通过较少的调整，利用Flutter Web搭建了整个Workflow的前端页面。

![Slide30.png](http://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/b746718c-9bc1-4dd3-a5d7-876451be952bSlide30.png)

图16 开发踩坑

在Flutter的工程实践上，需要投入大量的人力。长远来看，我们期望将**Magpie**打造为一个完整的**持续交付平台**。在开发阶段，测试阶段，开发部署，发布部署这四个关键环节都还有很多的工作等待我们。

![Slide33.png](http://wos.58cdn.com.cn/IjGfEdCbIlr/ishare/9570a9da-8b97-4672-b2b1-6cf38bbd5323Slide33.png)

图17 后续规划

## 5 小结

最后一点，58自研的**Magpie**解决方案是开源的。这意味着，如果你感兴趣，可以**拿来主义**用起来，过程中遇到任何问题，或建议，欢迎到github反馈。

> [https://github.com/wuba/magpie](https://github.com/wuba/magpie)

限于笔者对Flutter的研究深度，文中不正确之处，望不吝指正：）


## 作者简介

吴朝彬，58同城 Android资深开发工程师，主要负责58同城App的相关研发工作。