---
layout: post
title: "好用到飞起来的字符parameter substitution技巧"
description: "处理文本的使用技巧"
header_image: /assets/img/2017-01-04-01.jpg
keywords: "shell字符处理"
tags: [shell]
---
{% include JB/setup %}
![img](/assets/img/2017-01-04-01.jpg)

## 前言

在处理文本日志的时候，经常会需要对一些字符串做处理，比如分割特定块，路径匹配等等。

## 查找字符串

在过滤日志，源代码匹配时，经常会需要根据关键词检索目标文件;
比如在所有java代码中，查找“public View getView(int”，并将结果重定向到文件内

```
find . -name *.java | xargs grep "public View getView(int" >$logFile
```

## 切割字符串

当我们得到一个文件全路径后，有时会需要截取其中的路径部分，或者文件名部分，或者文件后缀，等等，如果是编程语言，很见到直接利用类似indexOf的接口即可，但是在shell怎么处理？

在shell中有一个parameter substitution的概念，正好可以处理我们这种情况

## 正则移除,前匹配

语法格式

```
${var#Pattern}          
${var##Pattern}
```
其中var是参数，一个#代表最短匹配，两个##代表最长匹配，Pattern是正则表达式，比如截取文件名，转换为表达式可以这样理解：$file是我们的路径，需要匹配最后一个/并去除匹配的文本，所以正则表达式就是*/

```
_filename="${file##*/}"
```

类似的如果仅截取后缀的话，只需要把/替换为.即可：

```
_expension="${file##*.}"
```
这个有个有趣的场景，就是在脚本中输出，当前脚本的名字

```
#!/bin/bash
_self="${0##*/}"
echo "$_self is called"
```

## 正则移除，后匹配
除了截取文件名，还可以截取文件路径，这个可以利用后匹配,语法格式如下：

```
${var%pattern}
${var%%pattern}
```
同理我们可以知道%的个数和#是一样的，所以，截取文件路径，也可以这样理解：$file是我们的路径，需要匹配第一个/并去除匹配的文本，所以正则表达式就是/*

```
_dirname="${file%/*}"
```

## 替换字符串

再来讲一下文本替换，类似于replace方法，也很好用，简单情形可以提点sed等工具,语法格式如下：

```
${varName/Pattern/Replacement}
${varName/word1/word2}
${os/Unix/Linux}
```
比如下面的语句可以替换unix并输出新的结果：

```
x="Ganji is cool, We use Ganji frequently"
echo "${x/Ganji/58}"
out="${x//Ganji/58}"
echo "${out}"
```

输出结果：

```
x="Ganji is cool, We use Ganji frequently"
echo "${x/Ganji/58}"
out="${x//Ganji/58}"
echo "${out}"
```
两句话区别在于，是否替换所有的关键词，由此我们知道多加一个/的作用是全部替换。

## 字符串截取

不同于简单的分割字符串，有时候我们还会需要精确的根据索引位置来截取子字符串，语法规则如下：

```
${parameter:offset}
${parameter:offset:length}
${variable:position}
var=${string:position}
```

参数可以有好几种写法，我们挑两个来分析，比如以下写法：

```
x="https://www.58.com"
echo ${x:8}
echo ${x:12:2}
```
输出结果为：

```
www.58.com
58
```
很简单，如果不指定长度那么会从开始位置一直截取到最后一个字符；

## 小结

上面这些技巧，都是现学现用，如果不常时候很容易忘记，因此做笔记就很有帮助，免去下次用到时疯狂Google的麻烦，最后把老外总结的一个表格贴出来，包括一些上面没提到的，方便整体回顾：

命令 | 说明 | 实例
------------ | -------------
${parameter:-defaultValue} | 参数默认值  
${parameter:=defaultValue} | 设置默认值
${parameter:?”Error Message”} | 如果参数不存在，显示错误信息
${#var} | 取字符串长度
${var%pattern} | 正则移除，后匹配，命中第一个  
${var%%pattern} | 正则移除 ，后匹配，命中所有 
${var:num1:num2} | 截取子字符串
${var#pattern} | 正则移除，前匹配，命中第一个  
${var##pattern} | 正则移除，前匹配，命中所有
${var/pattern/string} | 替换第一个
${var//pattern/string}| 替换所有

## 参考

* [http://www.tuicool.com/articles/6rqIzq](http://www.tuicool.com/articles/6rqIzq)
* [https://www.cyberciti.biz/tips/bash-shell-parameter-substitution-2.html](https://www.cyberciti.biz/tips/bash-shell-parameter-substitution-2.html)
