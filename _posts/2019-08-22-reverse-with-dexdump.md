---
layout: post
title: "逆向检测dex特征函数"
description: ""
header_image: /assets/img/2019-08-28-02.jpg
keywords: ""
tags: [dex]
---
{% include JB/setup %}
![img](/assets/img/2019-08-28-02.jpg)

* 目录
{:toc #markdown-toc}

## 背景
通过逆向工具我们可以反编译apk，进而查找一些关键词。但是这个效率往往比较慢，GUI工具占用内存特别大，而且查找也不是很方便。
本文分享下，怎么使用的dexdump快速检测代码。

## 查找特征代码
> 假设你现在需要检测一下apk中是否引入了某个sdk，或者是否有特定函数，

此时我们只需要分析出特征代码在编译后的方法签名即可快速查找，如果不严谨，甚至可以直接字符串匹配查查
简单的可以直接用linux下的strings命令，对zip递归查找。这里介绍下dexdump，分以下步骤

1. 解析出classesN.dex文件
2. 变量dex，针对每个dex执行dexdump，查找类定义，也就是dex中的class def
3. 根据命中方法，输出方法所在的类名
4. 输出报告

dexdump的使用，大家可以查阅帮助文档，默认该命令可能没有配置在PATH中，可以在以下路径找到
```shell
$ANDROID_HOME/sdk/build-tools/[platform版本号]/dexdump
```
通过 `-d`或`-h`参数都可以输出class列表定义，但是格式略有细微差异。

将上述流程编写为shell脚本方便后续使用。

### dexdump查找

为了能够使用dexdump，我们需要在安装了SDK的情况下自助发现dexdump的目录，这里我们查找最新平台版本的dexdump目录

```shell
if [[ -z "${ANDROID_HOME}" ]]; then
  die "ANDROID_HOME not found"
fi

echo "find target build-tools"
buildtools=($ANDROID_HOME/build-tools/*)
size=${#buildtools[@]}
buildtool=${buildtools[size-1]}
echo $buildtool
```
### dexdump输出分析

现在先分析下dex解析后的数据结构，可以参考下 [dex-format](https://source.android.com/devices/tech/dalvik/dex-format#header-item)

`-h`参数。首先输出的是head头信息不同字段的取值，接着就是class列表，这个是重点。
```
Opened '/Users/aven/Desktop/apk/classes.dex', DEX version '035'
Class #0 header:
class_idx           : 79
access_flags        : 17 (0x0011)
superclass_idx      : 9234
interfaces_off      : 0 (0x000000)
source_file_idx     : 24233
annotations_off     : 0 (0x000000)
class_data_off      : 7547075 (0x7328c3)
static_fields_size  : 0
instance_fields_size: 0
direct_methods_size : 1
virtual_methods_size: 0

```

我们看下class的定义，第一行是类名，接下来是类的各种定义信息，比如可见性，基类，接口继承，静态方法，成员方法。

因为我们是查找成员方法，因此关注的是Direct methods下的数组节点。
每个方法的输出，第一行是类名，第二行是方法名，另外就是是方法的属性信息。
```
Class #0            -
  Class descriptor  : 'Landroid/arch/core/R;'
  Access flags      : 0x0011 (PUBLIC FINAL)
  Superclass        : 'Ljava/lang/Object;'
  Interfaces        -
  Static fields     -
  Instance fields   -
  Direct methods    -
    #0              : (in Landroid/arch/core/R;)
      name          : '<init>'
      type          : '()V'
      access        : 0x10002 (PRIVATE CONSTRUCTOR)
      code          -
      registers     : 1
      ins           : 1
      outs          : 1
      insns size    : 4 16-bit code units
      catches       : (none)
      positions     : 
        0x0000 line=10
      locals        : 
        0x0000 - 0x0004 reg=0 this Landroid/arch/core/R; 
  Virtual methods   -
  source_file_idx   : 24233 (R.java)
```

观察完这段输出我们可以得到一个结论：
> 找到函数名后，往前一行输出，就是方法所在的类名

### dexdump数据解析

接下来水到渠成，我们看下核心的解析方法
```shell
dexdump(){
  match_lines=`$buildtool/dexdump -h $1 |grep -n $2|cut -d':' -f1`
  #count=`echo $match_lines|wc -l|tr -d '[:space:]'`
  if [ "$match_lines" != "" ]; then
	echo "### $1" >> ../report.md
    $buildtool/dexdump -h $1 > "dump-$1.txt"
	line_numbers=(${match_lines//\n/ })
	echo "found ${#line_numbers[@]} times" 
	for i in "${line_numbers[@]}"; do
	  #echo "===> suggetst class line @ $((i - 1))"
	  echo "\`\`\`" >> ../report.md
	  sed -n "$((i - 1)),${i}p" "dump-$1.txt" |cut -d':' -f2 >> ../report.md
	  echo "\`\`\`" >> ../report.md
	done
  fi
}
```

## 小结
理解dex数据结构，并且使用sdk的工具，可以方便的进行代码特征值的匹配。

## 参考
* [https://www.bugsnag.com/blog/dex-and-d8](https://www.bugsnag.com/blog/dex-and-d8)
* [https://source.android.com/devices/tech/dalvik/dex-format](https://source.android.com/devices/tech/dalvik/dex-format)
