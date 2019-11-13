---
layout: post
title: "buildSrc插件模板生成器"
description: "快速进行gradle插件开发，自动创建模板插件工程"
header_image: /assets/img/2019-11-13-01.png
keywords: ""
tags: []
---
{% include JB/setup %}
![img](/assets/img/2019-11-13-01.png)
* 目录
{:toc #markdown-toc}

看过Gradle Plugin教程的人应该都知道开发Gradle的Plugin插件有三种形式，适用范围略有差异：
1. build.gralde 文件内直接编写插件代码，简单快速，耦合比较大；
2. buildSrc 特定目录内编写插件代码，编译方便，但是IDE目前没有这种模板，无法自动生成；
3. 独立Gradle工程 完全解耦，可控性强，调试麻烦，每次修改都需要发布插件

在实际项目中这三种都会采用，**一般简单的gradle处理直接在脚本内编写，复杂且成体系的代码插件用独立工程管理**。
至于buildSrc用的较少。

## buildSrc插件模式
从实际开发角度来说，使用buildSrc最大的好处在于，兼顾随时修改的便捷性（同一个工程内）和脚本独立性；
特别是处于开发中的插件，需要大量调试和修改源码，如果每一次改动都需要发布版本，那是相当蛋疼的。

解决发布问题有些方法可以规避：
1. 发布到本地maven仓，避免污染远程仓库，
2. 发布为SNAPSHOT，避免频繁修改版本号

但是终究还是比较麻烦，如果采用buildSrc则可以愉快修改。

> 那么怎么创建一个buildSrc使得IDE能够识别它？

根据Gralde的说明[custom_plugins](https://docs.gradle.org/current/userguide/custom_plugins.html), 我们知道一个正确的buildSrc需要的格式目录如下：

![](/assets/images/buildsrc-tree.svg)

符合这个目录结构，gradle会自动识别buildSrc，并且在Gradle构建前先构建buildSrc下的插件代码：
![](/assets/images/gradle-buildsrc-plugin-log.png)

> 但是，如果只有手工创建的这些目录IDE是识别不了的😂

## IDE识别buidlSrc模块
为什幺让IDE识别模块？

其实主要是为了提高开发效率，代码提示和错误显示，语法高亮，这些的前提都是IDE能够正确的将工程源码识别。

Android Studio是基于IDEA的，识别一个目录需要`iml`配置文件，`iml`一般不提交到版本管理中。
由于开发IDE的不方便，笔者更习惯与使用IDEA开发Java和Gradle工程，使用AndroidStudio开发Andorid工程。
其实通过手工修改，可以在AndroidStudio中开发Gradle插件，大致步骤如下：
1. 新增一个Java Module，同时根据Gradle插件要求，命名为buildSrc
2. 将buildSrc这个模块内的文件，按照上述的工程结果手工修改

**PS**： 其实这里主要利用的就是IDE创建过程，让其为我们生成iml配置文件

## 模板工程
在AndroidStudio不支持自动创建Gradle Plugin的情况下，如何快速方便的为一个已有工程新增插件模块？

如果开发过IDEA的插件，可以考虑创建一个类似的New Module能力Plugin，但是开发Plugin的成本有点高，需要逆向分析借鉴IDEA的实现流程；完成插件之后还需要发布到Plugin市场上架。

另外一个就是手动创建一个模板，然后制作一套安装模板的脚手架，很多node的工程即使这个原理，比如npx创建react的脚手架工程。

所以要实现模板工程的方法是非常多的，笔者最后采取的方案是linux命令行安装。同时为了减少本地持久化文件的产生，我们借鉴了大名鼎鼎的Homebrew安装方案。整体流程比较简单，大致如下：

![](/assets/images/buildsrc-install-flow.png)

## 安装
基于设计思路，我们只做好安装脚本，然后上传到github仓库，需要适应时，执行一下脚本即可为当前工程快速创建插件模块：
```bash
echo "$(curl -fsSL https://raw.githubusercontent.com/hacktons/tpl/master/install-buildsrc)" | bash
```

更多脚本说明请移步至项目首页[https://github.com/hacktons/tpl](https://github.com/hacktons/tpl)

## 小结
有了模板工程后，可以很方便的对不同项目添加实验性的插件，再也不用花费时间去倒退IDEA，配置发布工作。
> 如果写一个IDEA的插件支持New Gradle Module会不会更炫酷呢？
