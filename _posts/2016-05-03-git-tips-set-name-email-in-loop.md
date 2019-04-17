---
layout: post
title: "批量设置git config"
description: ""
header_image: /assets/img/2016-05-03-01.jpg
keywords: ""
tags: [git]
---
{% include JB/setup %}
![img](/assets/img/2016-05-03-01.jpg)

## 前言
能偷懒就偷懒

## 需求场景
在开发新项目的时候经常需要配置git，在config中常用的有name和email，全局配置一般已经配过了，但是需要对特殊repo做个别处理，以确保提交的时候显示的作者是合乎规范的；  
最近在熟悉公司项目的时候，由于项目仓库特别多，大概10几个吧，如果每个repo都手动去设置user肯定要疯了，因此有了下面要讲的批处理脚本；

## 让脚本去做吧
针对这种情况，特别适合bash出手，我们要做的实际上就是遍历工程目录，对所有事仓库的目录都配置一下user, 为了友好一点，在输入不合法是提示用户正确的操作姿势：

{% highlight shell %}
#!/usr/bin/env bash
##############################################################################
##
##  set name and email required by git for all repositories in the directory
##
##############################################################################

if [ $# -ne 2 ]; then
    echo "Usage: $0 \"Name\" \"Email@58ganji.com\""
    exit 1
fi
user_name=$1
user_email=$2
for repo in ./*
do
    if [ -d $repo ]; then
        cd $repo
        if [ -d ".git" ]; then
            echo "Found new repo directory:$repo"
            git config user.name "$user_name"
            git config user.email "$user_email"
            echo "config done"
        fi
        cd ../
    fi
done
{% endhighlight %}

## 小结
没什么特别的技巧，活用shell可以在工作中更加便利
