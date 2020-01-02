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
通过pub或者flutter pub安装mpcli

```shell
pub global activate mpcli
```

安装dart全局命令行后，就可以开始使用它进行项目创建等后续流程了。

![](/assets/images/cli-workflow.png)

1.创建flutter模块工程

```shell
# 创建模板工程
mpcli create
```

2.启动workflow

```shell
# 进入新创建的工程目录内
cd your-project
# 启动workflow
mpcli start
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

## 小结
workflow即将内部开源，更多技术细节请请期待。

