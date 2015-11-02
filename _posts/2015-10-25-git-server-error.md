---
layout: post
title: "git server搭建异常中的处理"
description: ""
category: 
tags: []
---
{% include JB/setup %}

日常开发中经常使用git来获取代码，版本管理，对于git在服务器上的安装配置接触相对来讲很少；
本文旨在记录一下git server配种过程中常见的问题，以免其他同仁碰到相同问题时再费脑细胞；

照例直接去到git的官网查看doc，不知道网址的话google/bing一下

	git server
	
不出意外的话第一个就是.

##操作步骤

接下来就是跟着指南一步步配置，第一次看的话，务必认真阅读每一段文字，这有助于更深的理解操作的每一步；
偷懒的话也可以直接看我总结的步骤：

首先登陆服务器，或者本地的模拟账号，这很关键

1. git init --bare sample-project.git

		这里和平时新建一个仓库的唯一区别是--bare参数，sample-project.git故意带了git后缀
	
2.	~/.ssh/authorized_keys

		接下来时很熟悉的一个步骤，添加ssh登陆地支持，把用户的公钥（xxx.pub）以文本形式附加到authorized_keys里面
		
3. git clone/push/pull

		现在可以正常使用sample-project.git这个仓库了
		
##错误处理
由于每个人的电脑环境，配置信息不尽相同，可能会遇到各种各样的问题，下面是笔者在实践过程中遇到的几点错误；

1. git-receive-pack找不到，或者git-upload-pack 找不到

这类问题一般是路径问题，如果path已经配置了过了，而且直接在终端中是可以找到相关命令，那么可能是path配置的文件不合适,*nix系统下环境变量可以配置在很多中，常用的有用户目录下的.bashrc和.profile，具体区别自行google

2. 配置了ssh，但是提示权限问题

这个有可能是ssh生产公钥私钥时复制出现问题，如果公钥文本复制没问题，检查.ssh下是否有多套秘钥，如果有，需要用config配置好




		