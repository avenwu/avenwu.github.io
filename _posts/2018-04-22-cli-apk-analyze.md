---
layout: post
title: "安装包大小分析"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2018-04-24-01.png
keywords: ""
tags: [cli, apk]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2018-04-24-01.png)

我们已经介绍过如何使用Golang来开发脚本，并通过homebrew来发布。

在对安装包的分析的过程中发现一些比较有趣的事情，记录在此。

## APK大小

在真正分析安装包之前，有必要再讲一下大小问题，下面我们通过三个计算式开始：

* 文件大小
* 磁盘大小
* APK下载大小

按套路来说，这里应该只有一个磁盘大小和文件大小，为什么还有一个APK下载大小呢？

![apk-size](http://7u2jir.com1.z0.glb.clouddn.com/img/apk-size-1.png)

![apk-size](http://7u2jir.com1.z0.glb.clouddn.com/img/apk-size-2.png)

* 文件大小就是文件的真实字节数，比如2048bytes, 磁盘大小是指文件占用的空间；
* 大于文件的字节数，一般以1000进行换算；
* 下载大小指的是文件传输过程中的大小，一般小于等于文件大小，这取决于传输过程是否有进行二次压缩处理；

根据分析，我们可以知道，针对APK安装包存在3中大小，我们需要关心的是哪种呢？

> 文件大小和下载大小

Android Blog曾经出过一篇博文对这Google Play涉及的APK下载大小做过简单分析, 详见[Improvements for smaller app downloads on Google Play](https://android-developers.googleblog.com/2016/07/improvements-for-smaller-app-downloads.html)

并且由此我们得出一个结论：

> APK瘦身不要违反Android的设计意图，APK内的资源并不是刻意随意压缩的的，比如：*.so, resources.arsc

我们知道Play在升级应用时，会采用增量更新来减少所需要的下载的字节数，[bsdiff (created by Colin Percival1)](http://www.daemonology.net/bsdiff/)。
他的主要压缩点是app中的native代码，因此so在apk中最好是未压缩的stored形式。

这是一个示例数据：

| **Patch Description**   | **Previous patch size** | **Bsdiff Size** |
| ----------------------- | ----------------------- | --------------- |
| M46 to M47 major update | 22.8 MB                 | 12.9 MB         |
| M47 minor update        | 15.3 MB                 | 3.6 MB          |


```
Bsdiff is specifically targeted to produce more efficient deltas of native libraries by taking advantage of the specific ways in which compiled native code changes between versions. To be most effective, native libraries should be stored uncompressed (compression interferes with delta algorithms).
```

据说bsdiff算法比以前使用的增量算法压缩效果更好，然而这并不是本文关注的重点：

```
Apps that don’t have uncompressed native libraries can see a 5% decrease in size on average, compared to the previous delta algorithm.
```

另外使用了[xdelta算法](http://xdelta.org/)生成apk增量包。

因此在Google Play上上架的App需要关注`Downlaod Size`和`Update Size`，这两个数字是直接展示在Play Store中的，并且这两个值都不是APK自身的大小。

Chrome示例：

|                         | Compressed Native Library | Uncompressed Native Library |
| ----------------------- | ------------------------- | --------------------------- |
| APK Size                | 39MB                      | 52MB (+25%)                 |
| Download size (install) | 29MB                      | 29MB (no change)            |
| Download size (update)  | 29MB                      | 21MB (-29%)                 |
| Disk size               | 71MB                      | 52MB (-26%)                 |


## 安装包对比

基于Jenkins的自动化构建流程，我们可以实现对每次构建APK大小分析。从而形成统计图表。在分析版本迭代和不同产品分析时，自然也需要类似处理，这里我需要对比几个APK之间的大小差异，以及每个安装包中的几个核心块的大小变化。

先来看一下最终效果图：

![apk-size-line](http://7u2jir.com1.z0.glb.clouddn.com/img/apk-size-line.png)

![apk-version-details](http://7u2jir.com1.z0.glb.clouddn.com/img/app-compare.png)

一个apk标准的apk中必然会有以下几块内容：

* AndroidManifest.xml
* resources.arsc
* classesN.dex
* res/
* assets/
* lib/
* META-INF/

这些内容中，AndroidManifest.xml的大小基本可以忽略，除此之外还有一些额外文件，比如各种证书，配置文件，文本文件等，我们把这些非标准文件和小文件统一为others

安装包的分析主要就是利用zip读取apk，统计不同种类文件的大小，实现方法有很多，这里将基于Golang实现。

## 工具分析

分析apk需要确定：如何计算Download Size？开发者文档中并没有找到相关说明文档，除了前面博客中提到的生成增量包用xdelta算法。

通过分析，我们知道Android Studio和apkanalyzer在计算时采用的都是Gzip，并且使用了压缩强度最大的-9。

也就是对apk安装包再做一次gzip压缩，得到的新的gz文件，获取他的文件大小就是Download Size。
使用java nio里面的copy直接将一个Path代表的文件拷贝到目标文件Path, 代码非常简洁。

示意代码如下：

```java
try (GZIPOutputStream zos = new MaxGzipOutputStream(new FileOutputStream(compressedFile))) {
  Files.copy(apk.toPath(), zos);
  zos.flush();
}
catch (IOException e) {
  Logger.getInstance(ApkParser.class).warn(e);
  return apk;
}
```

对zip再做一次gzip，实际上压缩的比率是非常有限的，笔者测试的样例中压缩率大概是5.4%

在Golang里面类似，我们是这样处理的：

```go
func gzipFile(name string, dst string) (*os.File, error) {
	os.Remove(dst)
	apk, _ := os.Open(name)
	temp, _ := os.Create(dst)
	zw, _ := gzip.NewWriterLevel(temp, gzip.BestCompression)
	defer zw.Flush()
	defer zw.Close()
	io.Copy(zw, apk)
	return temp, nil
}
```

压缩完的大小非常接近，但是还是有点差异。

## 脚本使用

理清楚了前后思路和关键点，我们便可以打通流程，实现一个简单的工具。

```shell
brew install hacktons/cli/apkcompare
```
安装过程取决于网上，正常一分钟应该搞定了。

使用工具也是异常简单,直接看帮助文档好了

```shell
aven-mac-pro-2: aven$ apkcompare -h
Usage of apkcompare:
  --format string
    	foramt to output: json, xlsx(which is default) (default "xlsx")
  --log
    	Print all debug log
  -o, --output string
    	Output excel name (default "output.xlsx")
  -p, --path string
    	Path or Directory of the target apk file (default "./")
  --readable
    	output the size in MB, instead of bytes count (default true)
```

比如我在本地有一个test目录，里面有待测的若干apk，需要快速分析他们，可以这样做：

```shell
aven-mac-pro-2: aven$ apkcompare -p ~/Desktop/test -o ~/Desktop/result.xlsx
```

如此，便可以得到一个excel文档

![apkcompare-excel](http://7u2jir.com1.z0.glb.clouddn.com/img/apkcompare-excel.png)

## 参考

* [https://android-developers.googleblog.com/2016/07/improvements-for-smaller-app-downloads.html](https://android-developers.googleblog.com/2016/07/improvements-for-smaller-app-downloads.html)
* [https://developer.android.com/topic/performance/reduce-apk-size.html#reduce-code](https://developer.android.com/topic/performance/reduce-apk-size.html#reduce-code)
* [https://developer.android.com/studio/build/apk-analyzer.html](https://developer.android.com/studio/build/apk-analyzer.html)
* [https://developer.android.com/studio/command-line/apkanalyzer.html#syntax](https://developer.android.com/studio/command-line/apkanalyzer.html#syntax)
* [https://developer.android.com/studio/releases/sdk-tools.html](https://developer.android.com/studio/releases/sdk-tools.html)
* [https://github.com/360EntSecGroup-Skylar/excelize](https://github.com/360EntSecGroup-Skylar/excelize)
* [https://golang.org/pkg/encoding/json/](https://golang.org/pkg/encoding/json/)
* [https://www.jianshu.com/p/bee02c18b221](https://www.jianshu.com/p/bee02c18b221)
* [https://golangtc.com/t/50d6bf54320b527e6d00001a](https://golangtc.com/t/50d6bf54320b527e6d00001a)

