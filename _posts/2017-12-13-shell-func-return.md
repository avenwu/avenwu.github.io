---
layout: post
title: "Shell中处理方法返回值问题"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2017-12-15-01.jpg
keywords: "shell,返回值"
tags: [shell]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2017-12-15-01.jpg)

## 背景

通过shell编程，写一些工具批处理的时候，经常需要自定义函数。更复杂点的情况下，可能有需要返回一个值。
由于在shell的世界中，并不像其他编程语言，它不支持我们所熟悉的方法返回。本文一起总结一下如何优雅的解决返回值问题？

## 测试程序

我们一般通过`$?`来获取上一个语句的输出。看一下下面得测试语句：

新建`testReturn`脚本

```shell
returnString(){
	return $1
}

returnString $1
result=$?
echo "result=$result"
```

现在我们有一个testReturn的脚本，里面有一个returnString的方法，我们希望它能够直接返回我们输入的参数。
当我们分别以`hello`,`500`,`12`作为输入参数时，他的执行和输出情况是一样的么？

```
./testReturn hello
./testReturn 500
./testReturn 12
```

在心中试着猜一下可能的情况，现在我们来揭晓答案：

### 程序输出情况

在执行hello的时候，并没有输出hello，而是报了一个return只接受数字类型的错误

```
aven-mac-pro-2:avenwu.github.io aven$ ./testReturn hello
./testReturn: line 23: return: hello: numeric argument required
result=255
```

在执行500的时候，页没有输出500，而是输出了244

```
aven-mac-pro-2:avenwu.github.io aven$ ./testReturn 500
result=244
```

执行12的时候，终于正确了，返回12

```
aven-mac-pro-2:avenwu.github.io aven$ ./testReturn 12
result=12
```
## 异常分析

现在我们分析一下returnString这个方法，为什么会有这么多种输出情况呢？
首先他的写法显然是不严谨的，但也不是完全错误，比如输入12他就正确返回了。

`return`本身是shell里面的buildin函数，笔者总结了下，他有以下几个特征：

* return可以返回数字状态，常常用于返回0，1,标识一个函数执行后是否成功
* 注意return不可以返回非数字类型
* 同时数字类型也有可能发生溢出现象

## 全局变量

如果我们就是要返回一个字符串，怎么办呢？可以通过定义全局变量来进行赋值，类似于静态变量/成员变量的写法，我们让他的作用于穿透整个上下文。

```shell
result=""
returnString(){
	result=$1
}

returnString $1
echo "result=$result"
```

再看一下输出，得到了我们需要的结果：

```shell
aven-mac-pro-2:avenwu.github.io aven$ ./testReturn hello
result=hello
aven-mac-pro-2:avenwu.github.io aven$ ./testReturn 500
result=500
aven-mac-pro-2:avenwu.github.io aven$ ./testReturn 12
result=12
```
但是这样写，会污染全局变量，并且result这个变量很容易在内部和外部都被修改，导致内部修改失效。

## eval

除了`return`，还有其他一些buildin的关键字，比如`eval`，`local`。
默认在当前脚本定义的变量都是全局变量，在方法中则可以通过local来定义局部变量，这样可以避免全局变量污染.
同时结合eval赋值语句，来实现变量的修改

```shell
returnString(){
	local __result=$2
	eval $__result=$1
}

returnString $1 result
echo "result=$result"
```

同样我们也得到了希望的结果

```
aven-mac-pro-2:avenwu.github.io aven$ ./testReturn hello
result=hello
aven-mac-pro-2:avenwu.github.io aven$ ./testReturn 500
result=500
aven-mac-pro-2:avenwu.github.io aven$ ./testReturn 12
result=12
```

## echo

最后在介绍一种方法，通过`echo`输出，结合`command substitution`。
这个`command substitution`也没有找到比较合适的翻译，姑且按字面意思翻译`命令替换`。

如果你的方法内部只有一处echo输出，那么也可以利用她来进行值得返回，不过这个就有点局限性，一定要确保方法内只有一次输出，否则就会出现赋值内容过多。

```shell
returnString(){
	local __result=$1
	echo $__result
}
# 或者 result=`returnString $1`
result=$(returnString $1)
echo "result=$result"
```

同样可以得到预期结果

```
aven-mac-pro-2:avenwu.github.io aven$ ./testReturn hello
result=hello
aven-mac-pro-2:avenwu.github.io aven$ ./testReturn 500
result=500
aven-mac-pro-2:avenwu.github.io aven$ ./testReturn 12
result=12
```

## 越界问题

现在我们已经有几种办法可以返回字符串了，那么return返回数字有时候正确，有时候又不正确是为什么呢？

我们知道return原本就是用于返回执行状态的，比如0，1.那么我们在返回500的时候，实际上是数据溢出了。

根据测试，我们推断shell的内置return承接返回值用的是一个字节的大小，也就是8位，最多可以输出无符号0-255的整形，范围之外的数据全部溢出显示。因此在使用return的时候，务必留意数值大小。

## 小结

通过shell命令可以很方便的写出一些小脚本，但是如果遇到逻辑复杂，建议通过其他更合适的预览来实现，比如Python，Golang之类。

* [http://www.linuxjournal.com/content/return-values-bash-functions](http://www.linuxjournal.com/content/return-values-bash-functions)