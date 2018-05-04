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

## 压缩细节分析

我们知道早前Andresguard出来后，引领了一波对资源的压缩处理，包括resources.arsc混淆处理和本身的存储压缩。

那么在我们分析的对象中，各家app都是怎么处理的呢？

我们分析了一些装机必备的app，对比了他们在资源压缩上的差异，统计出如下表：

**Defl:N、Stored**为文件在zip中归档的存储方式，Stored表示未经压缩

|  主体  |                   SHA1                   | resources.arsc 压缩 | *.so压缩 | resources.arsc混淆 |  *.png压缩  |
| :--: | :--------------------------------------: | :---------------: | :----: | :--------------: | :-------: |
|  微信  | 6f90b4e0446f76d25aebd95f05e173c738e810dc |         否         |   是    |        是         |  0/1586   |
|  手Q  | 785af7f500d9138682dcaae1934e7f084ffacb03 |         否         |   是    |        是         |  2/6176   |
| 百度地图 | 18eee76c72f6bd948a7c4aaa5c979daa2a2efed0 |         否         |   是    |        否         |  0/5280   |
|  滴滴  | 01aad1fb6032efd1a6dccbd3d3e04ac9d149b1e1 |         是         |   是    |        是         | 2724/3733 |
|  美团  | 932986691bf889f72c331a9e53e61008341ab3fe |         是         |   是    |        是         | 6867/6916 |
|  携程  | c00e696210cb798444ff03c28ca4a6939b595cd3 |         是         |   是    |        否         | 125/1353  |
|  58  | 24119c50fa86eece2c053f2345d25fecfab84cb4 |         否         |   是    |        否         |  0/4108   |
|      |                                          |                   |        |                  |           |

通过对比，我们分析出以下线索：

* 微信研发的Andresguard被广泛使用，基本都对res做了命名的混淆；
* 微信和手Q本身没有压缩resources.arsc，是以Stored形式存在；
* 滴滴，美团都对png进行了压缩，应该有自己的一套干预打包的逻辑；
* 所以目标中的so都是以压缩形式存在；


实际上对arsc和so的处理是有争议的，如果是上架Google Play的包完全没必要自行压缩resources.arsc，so，png，这样反而会影响apk增量包的大小，已经apk安装后的大小。

原因可以参考`Andresguard`上的讨论和Google的文章：

* 如果不是对APK size有极致的需求，请不要把resource.asrc添加进compressFilePattern. (#84 #233)
* 对于发布于Google Play的APP，建议不要使用7Zip压缩，因为这个会导致Google Play的优化Patch算法失效. (#233)

apk中有些资源默认是没有被压缩的，比如so文件，resources.arsc本身，这是因为这些文件需要被系统加载，同时存在多进程使用的情况，如果需要频繁解压缩实际上对app的运行效率是打折扣的

```
I like the idea of having a toolbox that lets you further optimize your APK, but some of the "optimizations" offered by AndResGuard go against good practices:

there are good reasons why some files in the APK are not compressed, such as resources.arsc or native libraries (*.so/dll), so they can be read directly from APK at runtime. By offering/suggesting adding compression to them you in fact prevent optimization and degrade the experience at runtime. Ideally AndResGuard should not compress resources.arsc or at least add an explanation in the manual of why it's not a good idea.

recompressing the APK with 7zip makes little sense for APKs distributed via Play, as it might prevent some optimizations such as File-by-file patches, which greatly reduce download size for updates. Also the initial download from Play is compressed anyway, so this andresguard feature would only affect the APK size on disk and to a small degree. At the very least you shouldn't guide developers who distribute on Play to use 7zip recompression
```

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

