---
layout: post
title: "当Dart全栈遇上Flutter Workflow"
description: "利用dart开发flutter开发的工具平台"
header_image: /assets/images/workflow-design.png
keywords: ""
tags: [Flutter]
---
{% include JB/setup %}
![img](/assets/images/workflow-design.png)
* 目录
{:toc #markdown-toc}

> 本文为版权归属 **58 Magpie技术团队**，转载请注明出处

---

通过定制Flutter编译链工具，可以实现很多个性化的能力，甚至提供flutter tool本身不支持的功能。
同时借力Dart全栈，可以搭建完整的前后端开发工具。

## Flutter上手
上手Flutter比较简单，走官方的接入规则，我们可以快速实现Flutter的接入和开发。但是这里面也会存在一些问题，比如：
* 端源码的重度耦合，Flutter与Native源码出现依赖关系
* Flutter编译链侵入，增加Native开发，调试成本
* 同时运行多个 Flutter 实例或在部分视图中运行实例可能会导致未定义行为。
* 在后台模式下使用 Flutter 尚处于开发阶段。
* 不支持将一个 Flutter 库打包到另一个可分享的库内，或者将多个 Flutter 库打包到同一个应用中。

在大型工程中接入Flutter，势必需要考虑上述这几个问题。为了便于Flutter的接入，我们尝试统一接入标准，简化成本，我们的预期是这样的：
> 1. 通过提供定制的SDK,实现Android，iOS的Flutter统一接入
2. 通过提供开发接入的workflow，实现内部标准化，降低成本

## 奔跑吧workflow
通过workflow实现另一种flutter模块的创建，开发，编译，打包流程。

![](/assets/images/workflow-design.png)

在安装使用workflow前需要确保您本地的开发环境已经搭建完毕，并配置了相关的环境变量。
```
export PATH=/*flutter directory*/flutter/bin:$PATH
```
同时建议配置`dart sdk`的环境变量，方便`pub`的使用
```
export PATH=/*flutter directory*/flutter/bin/cache/dart-sdk/bin:$PATH
```

接下来就可以安装命令行工具，方便部署和启动workflow平台：
通过pub或者flutter pub安装mgpcli

```shell
pub global activate mgpcli
```

安装dart全局命令行后，就可以开始使用它进行项目创建等后续流程了。

![](/assets/images/cli-workflow.png)

1.创建flutter模块工程

```shell
# 创建模板工程
mgpcli create
```

2.启动workflow

```shell
# 进入新创建的工程目录内
cd your-project
# 启动workflow
mgpcli start
```
3.进入workflow

**现在已经为您打开了一个浏览器窗口**，可以移步至窗口进行：[编译、Attach](http://127.0.0.1:8080)

![](/assets/images/workflow-preview.png)

## Dart闭环开发
在整个workflow项目中，我们采取了分层架构设计，具体来说包括以下
![](/assets/images/WorkflowArchitecture.png)

在这一节中我们主要介绍前后端部分的架构设计。

前端页面基于Flutter-Web技术栈。开发Flutter的dart页面，编译时利用dart2js，将Flutter页面转为js，整体来说是一个SPA的单页面应用。

![](/assets/images/flutter-web.png)

后端部分与Flutter基本没有关系，属于纯Dart工程，基于Dart Server开发的后端服务。

有了前后端工程之后，我们做到了业务上的隔离。具体开发的时候，既可以隔离开发，也可以通过组织结构，实现同IDE开发前后端。

在实践中为了减少沟通成本和打通Dart全栈，参与开发的RD既要负责各自模块的前端业务，同时要考虑后端接口开发。实现了提个模块的的闭环开发。

![](/assets/images/workflow-structure.png)

## 编译介绍
**0x1 cli编译**  
命令行源码位于`cli`下，是一个Dart Cmdline工程，推荐通过VS Code进行开发。

**0x2 后端编译**  
workflow分为前后端，后端源码位于`server`下，是一个Dart工程，推荐通过VS Code进行开发。

**0x2 前端编译**  
前端部分是一个flutter工程，采用了flutter-web技术栈，可以通过`VS Code`, `Android Studio`开发。

为了顺利联调，建议先启动workflow的后端部分，启动可以通过
> ./start-server

后端目前不支持热重载，如果您修改了server代码，需要重启server，也就是重新执行`./start-server`。

启动web可以走IDE，也可以走命令行：
 > flutter run -d chrome

**0x3 编码规范**  
Dart编码遵守`Effective Dart`准则。项目已开启lint检测，不符合准则的代码，开发期间有会提示`warning`。

`Effective Dart`完整规范，请查看 [https://dart.dev/guides/language/effective-dart/style](https://dart.dev/guides/language/effective-dart/style)

集成发布意味着把前后端workflow整体编译打包，用户可以通过mgpcli进行使用。

## 集成发布
**发布workflow**  
执行以下命令完成前后端资源的构建和内置：
> ./deploy -cli

**本地验证cli**  
通过pub --source进行本地部署，执行以下命令：
> ./setup-cli

如果你本地有dart、flutter环境，但是不支持flutter-web，也可以使用预编译的workflow进行体验：
> ./setup-cli -dry-run

完成上述步骤后，你应当可以使用mgpcli命令了。例如启动workflow可以执行：
> mgpcli start

## 脚手架cli
在前面各小结中频繁出现的cli/mgpcli是我们的命令行工具。作为Workflow启动和门户，非常重要。
>面向用户下载安装的mplci是我们发布的后的工具。那么如何在本机进行脚手架开发和环境控制呢？

本机运行开发cli，我们定义为`Debug 模式`:

> Debug 模式为Magpie开发小组日常开发环境，感兴趣的同学也可以参与Magpie团队一起开发，大家一起创造更好的Flutter的开发环境。

具体操作如下：
* cli本地部署
```
git clone git@igit.58corp.com:flutter-lab/magpie_workflow.git 
pub global activate --source path /*本地全路径*/magpie_workflow 
```

* 源码运行环境配置
>1、进入用户根目录下的.pub-cache/bin/mgpcli  
>2、vi 打开mgpcli脚本文件我们可以看到dart "**/**/.pub-cache/global_packages/mgpcli/bin/mgpcli.dart.snapshot.dart2" "$@"  
>3、我们可以把打开文件中dart后面的路径指向我们clone下来的源码路径"**/magpie_workflow/cli/bin/mgpcli.dart"  
>4、用vscode打开工程，定位到mgpcli.dart文件，右键选择运行。至此Magpie工程就可以断点Debug了s

当前我们的脚手架支持的基础指令如下：

![](/assets/images/mgpcli-cmd.png)

## 小结

Workflow系列文章：

* iOS Magpie SDK实现
* Android Magpie SDK实现

workflow即将内部开源，更多技术细节请请期待。
