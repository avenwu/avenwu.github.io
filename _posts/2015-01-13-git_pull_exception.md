---
layout: post
title: "RPC failed result=18, HTTP code =200| 1024 bytes/s"
description: "git pull 异常"
category: 
tags: [git]
---
{% include JB/setup %}
在使用git管理项目代码时非常方便，由于经常在不同计算机上开发项目，pull代码时突然报异常：  

	RPC failed result=18, HTTP code =200| 1024 bytes/s
	fatal: The remote end hang up unexceptedly
	fatal: early EOF
 	fatal: index-pack failed


![Project Structure](http://7u2jir.com1.z0.glb.clouddn.com/git_pull_remote_hang_exception.png)  
Google找到了很多类似问题，大致说的都是传输内容超出默认限制，需要将限制放宽，如：  

	git config --global http.postBuffer 524288000

这里的524288000估计是byte为单位等于500M。但是即使放宽了限制仍然没有解决我遇到的问题，异常仍然在。
实在没办把http地址换成用ssh的，完美解决。