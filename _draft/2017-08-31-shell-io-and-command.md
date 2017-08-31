---
layout: post
title: "让你的批处理提示更直观"
description: "shell输出流与检测命令是否存在"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-04-18-02.jpg
keywords: "shell"
tags: [shell, io]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-04-18-02.jpg)

# 前言

在写脚本做批处理任务时，或多或少会使用到一些常规发行包没有安装命令，或者在macOS下开发的脚本要运用到linux/unix的各种变种时可能会遇到类似问题；
因此有必要在使用非常规命令前，做一下检测；

# IO流
下面介绍一下如何在使用命令

# 检测命令可用性

通过google，可以发现一些通行的写法, 比如我们想检测当前是否存在jq命令，如果不存在输出警告信息，并终止运行：

```shell
type jq >/dev/null 2>&1 || { echo >&2 "jq is not installed, please check out: https://stedolan.github.io/jq/"; exit 1;}
```

在首次看到这段代码时，有点一头雾水，比如`/dev/null`是什么意思，`2>&1`又是什么意思；
实际上这个事涉及shell的输入输出流，在标准情况下shell有一些约定的变量。

## /dev/null

/dev/null 实际上是一个设备符，根据linux系统的说明，/dev下面都是一些设备符，而/dev/null就是讲输出重定向到空设备，也就是忽略输出；

## 2>&1

这里的2实际是表示错误输出，1是标准输出，因此这段句话的意思执行脚本的错误输出重定向到标准输出


# 参考
https://www.topbug.net/blog/2016/10/11/speed-test-check-the-existence-of-a-command-in-bash-and-zsh/
https://stackoverflow.com/questions/592620/check-if-a-program-exists-from-a-bash-script
https://stackoverflow.com/questions/7522712/how-to-check-if-command-exists-in-a-shell-script
