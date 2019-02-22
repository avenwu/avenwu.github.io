---
layout: post
title: "Android P非SDK接口的限制分析"
description: ""
header_image: "/assets/images/android-p-clear-bg-with-shadow.png"
keywords: "Android P适配"
tags: [Android]
---
{% include JB/setup %}
![img](/assets/images/android-p-clear-bg-with-shadow.png)

## 背景

Android系统几乎每年都在发布新版本，在Android 9上，针对非SDK接口使用的警示需要开发者提前考虑。

```
Android 9（API 级别 28）引入了针对非 SDK 接口的使用限制，无论是直接使用还是通过反射或 JNI 间接使用。 无论应用是引用非 SDK 接口还是尝试使用反射或 JNI 获取其句柄，均适用这些限制。 有关此决定的详细信息，请参阅通过减少使用非 SDK 接口提升稳定性。

一般来说，应用应当仅使用 SDK 中正式记录的类。 特别是，这意味着，在您通过反射之类的语义来操作某个类时，不应打算访问 SDK 中未列出的函数或字段。

使用此类函数或字段很可能会破坏您的应用。
```
## 静态检测

如果你的应用使用了限制接口，在调试包运行时，会弹出警告框，按图索骥，核查一下官方说明：[https://g.co/dev/appcompat](https://developer.android.com/about/versions/pie/restrictions-non-sdk-interfaces)

Google提供了静态检测工具[veridex](https://android.googlesource.com/platform/prebuilts/runtime/+/master/appcompat)，用于检测apk是否使用了限制api。

### veridex使用
该工具是一个*nix下的二进制可执行脚本，可以在macOS和linux下使用，Google官方已经提供了一套预编译好的脚本，如果有linux开发环境，可以阅读README.md执行响应入口脚本:

```shell
./appcompat.sh --dex-file=test.apk
```

如果执行失败，需要确认一下环境问题。

笔者的本地为macOS环境，除此遇到了执行失败，日志类似：

```
NOTE: appcompat.sh is still under development. It can report
API uses that do not execute at runtime, and reflection uses
that do not exist. It can also miss on reflection uses.
./appcompat.sh: line 28: /Users/aven/Desktop/runtime-master-appcompat/veridex: cannot execute binary file
./appcompat.sh: line 28: /Users/aven/Desktop/runtime-master-appcompat/veridex: Undefined error: 0
```

分析日子可知veridex文件无法执行，主要原因是系统不兼容，他是linux下的可执行文件，不是macOS的。

```shell
aven-mac-pro-2:runtime-master-appcompat aven$ file veridex
veridex: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.24, with debug_info, not stripped
```

解决办法有两个，一是自行编译veridex，这个成本比较大，需要安装系列的编译依赖库。
第二个办法是，搭建一套可用的linux环境，这个相对简单。

如果你有在使用Docker，可以尝试已经存在一些镜像，如果没有，直接去Docker Hub找一个基于linux 2.6.24以上的镜像，然后在docker镜像内执行veridex即可

```
#132: Reflection greylist Lsun/misc/Unsafe;->allocateInstance use(s):
       Lcom/google/gson/internal/UnsafeAllocator;->create()Lcom/google/gson/internal/UnsafeAllocator;
       Lcom/networkbench/com/google/gson/internal/h;->sf()Lcom/networkbench/com/google/gson/internal/h;
       Lcom/wuba/certify/x/ad;->a()Lcom/wuba/certify/x/ad;

#133: Reflection greylist Lsun/misc/Unsafe;->theUnsafe use(s):
       Lcom/google/gson/internal/UnsafeAllocator;->create()Lcom/google/gson/internal/UnsafeAllocator;
       Lrx/internal/util/unsafe/UnsafeAccess;-><clinit>()V
       Lcom/networkbench/com/google/gson/internal/h;->sf()Lcom/networkbench/com/google/gson/internal/h;
       Lcom/wuba/certify/x/ad;->a()Lcom/wuba/certify/x/ad;

133 hidden API(s) used: 24 linked against, 109 through reflection
       126 in greylist
       2 in blacklist
       3 in greylist-max-o
       2 in greylist-max-p
```
检测结果将列出名字黑名单，灰名单，深灰名单的各种情况。后面就是依据不同限制名单注意核对源码作调整。

## 检测分析

前面我们根据官方提示，下载了脚本工具的压缩包，里面包含了很多资源。结合appcompact.sh的内容，在调用veridex时传递了很多参数:

```shell
# First check if the script is invoked from a prebuilts location.
SCRIPT_DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"

if [[ -e ${SCRIPT_DIR}/veridex && \
      -e ${SCRIPT_DIR}/hiddenapi-flags.csv && \
      -e ${SCRIPT_DIR}/org.apache.http.legacy-stubs.zip && \
      -e ${SCRIPT_DIR}/system-stubs.zip ]]; then
  exec ${SCRIPT_DIR}/veridex \
    --core-stubs=${SCRIPT_DIR}/system-stubs.zip:${SCRIPT_DIR}/org.apache.http.legacy-stubs.zip \
    --api-flags=${SCRIPT_DIR}/hiddenapi-flags.csv \
    $@
fi
```

### hiddenapi-flags.csv

其中有个excel文件，打开可以看到是一个白名单清单列表，配置了哪些特征值属于哪个级别的列表：

![img](/assets/images/android-p-white-list.png)

### org.apache.http.legacy-stubs.zip

另外还有一个`org.apache.http.legacy-stubs.zip`，里面是一个classes.dex,在尝试反编译后失败了:

```shell
aven-mac-pro-2:runtime-master-appcompat aven$ /Users/aven/Android/sdk/build-tools/28.0.3/dexdump classes.dex 
Processing 'classes.dex'...
E/libdex  (53872): ERROR: unsupported dex version (30 33 39 00)
E/libdex  (53872): ERROR: Byte swap + verify failed
ERROR: Failed structural verification of 'classes.dex'
```

根据[DexFormat.java](https://android.googlesource.com/platform/dalvik/+/master/dx/src/com/android/dex/DexFormat.java)的定义，Dex 039是Android 9使用的

```java

/** dex file version number for API level 28 and earlier */
public static final String VERSION_FOR_API_28 = "039";
/** dex file version number for API level 26 and earlier */
public static final String VERSION_FOR_API_26 = "038";
/** dex file version number for API level 24 and earlier */
public static final String VERSION_FOR_API_24 = "037";
/** dex file version number for API level 13 and earlier */
public static final String VERSION_FOR_API_13 = "035";
```

虽然Android自带的dexdump无法解析这个dex，但是jadx任然可以，升级到jadx 0.9.0

![dex](/assets/images/appcompact-dex.png)

分析可知，实际上这个dex是一些类的集合，这些类和老版本中sdk具备的API同名，但是都没有实现具体逻辑，推测功能主要是为了类检测校验之类，本身也不是为了被使用，因此不需要引入实现体，如此可以避开实现体内引用其他类的情况。

```java
public FilePart(String name, PartSource partSource, String contentType, String charset) {
    String str = (String) null;
    super(str, str, str, str);
    throw new RuntimeException("Stub!");
}
```

### system-stubs.zip
类似org.apache.http.legacy-stubs.zip，这个里面也是一个classes.dex

## 相关链接

* [https://developer.android.com/about/versions/pie/restrictions-non-sdk-interfaces#about-api-lists](https://developer.android.com/about/versions/pie/restrictions-non-sdk-interfaces#about-api-lists)
* [https://android.googlesource.com/platform/prebuilts/runtime/+/master/appcompat](https://android.googlesource.com/platform/prebuilts/runtime/+/master/appcompat)
* [https://android.googlesource.com/platform/frameworks/base/+/master/config/hiddenapi-greylist-max-o.txt](https://android.googlesource.com/platform/frameworks/base/+/master/config/hiddenapi-greylist-max-o.txt)
