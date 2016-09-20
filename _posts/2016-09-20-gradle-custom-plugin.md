---
layout: post
title: "Gradle基础进阶 Gradle插件"
description: "Gradle基础进阶 第二章 Gradle插件"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-07-31-01.png
keywords: "Gradle Beyond the Basics"
tags: [Gradle, Grade进阶]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-07-31-01.png)

## 前言

本文译自《Gradle Beyond the Basics》Chapter 2. Custom Plug-ins.

译文仅作学习交流，喜欢的朋友请支持正版。

* 正版书籍：[http://shop.oreilly.com/product/0636920019923.do](http://shop.oreilly.com/product/0636920019923.do)
* 全书中文翻译：[http://gradle-beyond-the-basics.avenwu.net/](http://gradle-beyond-the-basics.avenwu.net/)

## 正文

编写这样的构建任务是一种特殊的软件开发形式。问题核心并不在于项目的业务逻辑本身，而在于项目的构建自动化。这些特定的代码也是软件，Gradle正是用于解决这一需求的良药。要写这样的代码，未经训练的Gradle开发者可能会简单的在doLast中书写大量的Groovy语句。这样的代码会很难测试，并且会导致整个构建文件变得臃肿且不可测。因此这种做法是不建议的。为了解决这个问题，我们提供了插件API。

## 插件API

Gradel的插件指的是一个独立的归档文件，他继承了核心的Gradle功能。插件可以通过三种方式增强Gradle。第一种，插件本身可以直接接触Project对象，就像是一个额外的build文件混入了当前的build文件。Tasks，SourceSets，dependencies，repositories等等都可以被插件添加或者修改。

第二种，插件引入新的module到构建任务中，用于做一些特定的工作。插件以Java web service的注解形式创建WSDL文件时，不可以包含自己用于扫描注解，生成xml内容的代码。但是可以声明一个依赖关系来完成这个任务，这个依赖是一个现有的库，可以是本地的也可以是线上的仓库。

第三种，插件可以引入新的关键词和领域对象到Gradle构建语言当中。在标准的DSL中，没有描述发布必须指定的服务器位置，没有应用使用的数据库协议，源码控制工具可以执行的操作。相反，标准的DSL不能遇见到每一种使用情形已经开发者会遇到的领域。Gradle提供了完备的API文档，允许开发者基于实际需要自行扩展Gradle的标准构建语言。这Gradle的作为一个构建工具的核心优势。他允许我们编写简洁，声明式的构建任务。

## 插件案例

在这一节中，我们会创建一个Gradle插件，用于自动调用名为Liquibase的开源数据库重构工具。Liquibase是用Java编写的命令行工具，用于管理关系型数据库协议。他可以将一个已经存在的数据库协议逆向工程为一个XML的的变更日志，并且记录数据库变化的版本号，已决定是否需要对新的数据库执行重构。对应习惯Groovy语法的开发者们也可以选择使用Groovy Liquibase DSL

更多的的Liquibase信息可以在[Liquibase Quick Start](http://www.liquibase.org/quickstart)查阅。

Liquibase可以很好地实现他的功能，但是在使用上面，如果没有一个封装好的脚本，会比较笨重。另外，高度自动化的构建和发布通常是一个非显式的目标，我们倾向于把Liquibase的操作整合到构建的生命周期中。

在这一节汇总，我们的目标是完成下面这些事项：

* 创建一个Gradle的task，与generateChangeLog,changeLogSync和update这几个命令对应；

* 使用Groovy DSL配置默认的XML日志格式；

* 把这个Gradle的task重构为自定义的task类型；

* 引入Gradle DSL extensions来描述Changelogs和数据库配置；

* 将插件打包为独立的JAR文件；

我们的Liquibase插件奖惩标准的Gradle build文件开始编写。这种方式相对来说更容易编写原型代码，这也是开发新的自动构建任务时典型的工作流程。当插件开始形成我们想要的东西是，在慢慢将其重构为独立的插件项目。这种方式开发插件非常合适，并且降低了学习API的成本。


