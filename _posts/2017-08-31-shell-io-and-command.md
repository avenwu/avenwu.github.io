---
layout: post
title: "编写清晰的脚本：IO输出也能干大事"
description: "shell输出流与检测命令是否存在"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2017-08-31-04.jpg
keywords: "shell"
tags: [shell, io]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2017-08-31-04.jpg)

# 前言

在写批处理任务时，或多或少会使用到一些常规发行包没有安装命令，或者在macOS下开发的脚本要运用到linux/unix的各种变种时可能会遇到类似问题；
因此有必要在使用非常规命令前，做一下检测；这里就会用到shell输出的重定向。


# 检测命令可用性

通过google，可以发现一些通行的写法, 比如我们想检测当前是否存在jq命令，如果不存在输出警告信息，并终止运行：

```shell
type jq >/dev/null 2>&1 || { echo >&2 "jq is not installed, please check out: https://stedolan.github.io/jq/"; exit 1;}
```

在首次看到这段代码时，有点一头雾水，比如`/dev/null`是什么意思，`2>&1`又是什么意思；
实际上这个事涉及shell的输入输出流，在标准情况下shell有一些约定的变量。下面我们一步步来分析下这个过程。

# 标准输出

先来讲个简单的例子；我们知道 `>` 可以做输出的重定向， 比如将左侧内容输出到右侧：执行一个命令，并将结果输出到文件。

```shell
echo "hi std io" > test.txt
```

```shell
aven-mac-pro-2:avenwu.github.io aven$ ls -al test.txt 
-rw-r--r--  1 aven  staff  10 Sep  1 10:06 test.txt
aven-mac-pro-2:avenwu.github.io aven$ cat test.txt 
hi std io

```
在这个过程中其实有一个背后的约定，那就是重定向时的一些默认值，`>`本身只是重定向输出，具体将什么东西重定向是有开发者决定的；根据约定标准输出stdout用1表示，标准错误输出stderr用2表示，当然标准输入就是0了;

默认情况下`>`等价于`1>`,所以上面的语句实际上也等同于:

```shell
echo "hi std io" 1> test.txt
```

那如果我写成这样会使什么结果

```shell
echo "hi std io" 2> test.txt
```
实际上这样的话test.txt文件内什么内容都没有，是空的，但是屏幕上会有内容；这句话的意思是执行echo输出内容，同时将错误输出重定向到test.txt文件中；
我们知道这echo "hi std io"没有语病是可以正常执行的，所以不会有错误内容，所以test.txt也就没有内容，但是标准输出没有指定重定向，所以还是会输出到默认的屏幕上；

# 合并输出流

我们可以分别操作stderr和stdout, 同样可以合并两个输出，`m>&n` 表示将m代表的输出句柄与n代表的句柄合并，常见的 `2>&1` 就是讲标准错误输出和标准输出合并。

## /dev/null

/dev/null 是一个设备符，指向/deve/null 也就是忽略输出内容，比如本文接扫的检查命令是否存在，实际上就是忽略所有输出和错误输出的内容，至关心返回值，就可以判断状态；

所以`type jq >/dev/null 2>&1`的返回值是0或者1

# 小结

到这里，整个语句的含义已经分析完毕;

* 首先通过type来检验一个命令的可用性，当然也有其他很多类似的命令可以做这件事
* 同时将将错误输出与输出合并
* 最后将内容重定向到/dev/null，我们不需要文案信息，只需要返回值对应的状态
* 更加返回值，如果是非成功(非0),那么执行一段语句，输出可读性较强的说明信息，比如提示用户，当前确实某个命令，可以去哪里安装等；

```shell
type jq >/dev/null 2>&1 || { echo >&2 "jq is not installed, please check out: https://stedolan.github.io/jq/"; exit 1;}
```

# 参考

* [https://www.tutorialspoint.com/unix/unix-io-redirections.htm](https://www.tutorialspoint.com/unix/unix-io-redirections.htm)
* [https://www.topbug.net/blog/2016/10/11/speed-test-check-the-existence-of-a-command-in-bash-and-zsh/](https://www.topbug.net/blog/2016/10/11/speed-test-check-the-existence-of-a-command-in-bash-and-zsh/)
* [https://stackoverflow.com/questions/592620/check-if-a-program-exists-from-a-bash-script](https://stackoverflow.com/questions/592620/check-if-a-program-exists-from-a-bash-script)
* [https://stackoverflow.com/questions/7522712/how-to-check-if-command-exists-in-a-shell-script](https://stackoverflow.com/questions/7522712/how-to-check-if-command-exists-in-a-shell-script)
