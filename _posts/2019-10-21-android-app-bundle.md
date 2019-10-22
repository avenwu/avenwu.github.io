---
layout: post
title: "Android App Bundle扫盲"
description: ""
header_image: /assets/img/2019-10-22-01.png
keywords: ""
tags: []
---
{% include JB/setup %}
![img](/assets/img/2019-10-22-01.png)

* 目录
{:toc #markdown-toc}

## 背景
Google Play支持以aab格式上传应用包，结果Android 5.0系统支持多apk的安装，aab方案成为了官方支持的一种模块动态化解决方案。

下面基于Google的文档信息，整理了aab需要了解的概念。

## AAB用途
> The Android App Bundle is a new upload format that includes all your app's compiled code and resources,
 but defers APK generation and signing to Google Play.
Google 开发的一种新的用于Google Play市场的文件上传格式，包含代码和资源，开发者无需手动针对不同维度上传split apk

## AAB环境要求
Android Studio 3.2+，Android 5.0(API level 21)以上支持AAB动态分发，
Android 4.4 (API level 20)及以下需要开启fusing能力，即生成并安装完整apk

## AAB Dynamic Delivery动态分发
基于aab压缩包，Play可以根据目标设备生成差异化的apk安装包（如设备密度xh/xxh, 语言，cpu架构），
用户可以获得更小的，精简的安装包

## Dynamic feature Module动态功能模块

借助动态分发，独立的功能模块可以在运行期间动态下发安装，无需在apk下载安装时即包含

## split APKs的类型、组成
* Base apk 基础包，包含基础能力，可作为其他split apk的依赖库被引用，初始安装即携带
* Configuration apks 包括Local语言， Screen density屏幕密度，CPU architecture架构
* Feature apks 非安装时必要的功能，可以使用时动态下发的其他业务模块apk

## aab split配置
在andorid模块呢，配置需要split的维度，实际打bundle时默认不写也可以生成
```gradle
bundle {
   language {
       enableSplit = true
   }
   density {
       enableSplit = true
   }
   abi {
       enableSplit = true
   }
}
```

## bundletool作用
用于测试Android App Bundle，也可以通过Google Play后台上传测试，都是用bundletool

## bundletool生成apks操作
```
bundletool build-apks --bundle=<path to .aab> --output=<out.apks>
```

只需要两个参数，--bundle表示aab的路径，--output表示名，产出也是一个zip包，将生成一个全量的apks，也可以指定设备生成apks，参数为--connected-device，如果有多少连接设备，通过--device-id=serial-id指定设备号

示例：
```
java -jar ~/Downloads/bundletool-all-0.3.3.jar build-apks --bundle=./app/build/outputs/bundle/debug/bundle.aab --output=out.apks
```

全量apks内部结构
```
aven-mac-pro-2:debug aven$ unzip -l out.apks 
Archive:  out.apks
  Length      Date    Time    Name
---------  ---------- -----   ----
  2579975  01-01-1980 00:00   standalones/standalone-hdpi.apk
  2617461  01-01-1980 00:00   standalones/standalone-tvdpi.apk
  2573656  01-01-1980 00:00   standalones/standalone-ldpi.apk
  2573159  01-01-1980 00:00   standalones/standalone-mdpi.apk
     6480  01-01-1980 00:00   splits/dynamicfeature-ldpi.apk
     6482  01-01-1980 00:00   splits/dynamicfeature-mdpi.apk
     6474  01-01-1980 00:00   splits/dynamicfeature-hdpi.apk
     6477  01-01-1980 00:00   splits/dynamicfeature-xhdpi.apk
     6484  01-01-1980 00:00   splits/dynamicfeature-xxhdpi.apk
     6477  01-01-1980 00:00   splits/dynamicfeature-xxxhdpi.apk
     6461  01-01-1980 00:00   splits/dynamicfeature-tvdpi.apk
    16223  01-01-1980 00:00   splits/dynamicfeature-master.apk
    14786  01-01-1980 00:00   splits/base-ldpi.apk
    15536  01-01-1980 00:00   splits/base-mdpi.apk
    52492  01-01-1980 00:00   splits/base-hdpi.apk
    57723  01-01-1980 00:00   splits/base-xhdpi.apk
    69091  01-01-1980 00:00   splits/base-xxhdpi.apk
    74011  01-01-1980 00:00   splits/base-xxxhdpi.apk
    59938  01-01-1980 00:00   splits/base-tvdpi.apk
  2601500  01-01-1980 00:00   standalones/standalone-xxxhdpi.apk
  2596585  01-01-1980 00:00   standalones/standalone-xxhdpi.apk
     9530  01-01-1980 00:00   splits/base-ca.apk
     9472  01-01-1980 00:00   splits/base-da.apk
  2585227  01-01-1980 00:00   standalones/standalone-xhdpi.apk
     9735  01-01-1980 00:00   splits/base-fa.apk
     9550  01-01-1980 00:00   splits/base-ja.apk
     9923  01-01-1980 00:00   splits/base-ka.apk
     9707  01-01-1980 00:00   splits/base-pa.apk
    10163  01-01-1980 00:00   splits/base-ta.apk
     9471  01-01-1980 00:00   splits/base-nb.apk
     9740  01-01-1980 00:00   splits/base-be.apk
     9556  01-01-1980 00:00   splits/base-de.apk
    10133  01-01-1980 00:00   splits/base-ne.apk
    10034  01-01-1980 00:00   splits/base-te.apk
     9470  01-01-1980 00:00   splits/base-af.apk
     9772  01-01-1980 00:00   splits/base-bg.apk
     9771  01-01-1980 00:00   splits/base-th.apk
     9508  01-01-1980 00:00   splits/base-fi.apk
     9933  01-01-1980 00:00   splits/base-si.apk
     9878  01-01-1980 00:00   splits/base-hi.apk
     9599  01-01-1980 00:00   splits/base-vi.apk
     9705  01-01-1980 00:00   splits/base-kk.apk
     9799  01-01-1980 00:00   splits/base-mk.apk
     9581  01-01-1980 00:00   splits/base-sk.apk
     9722  01-01-1980 00:00   splits/base-uk.apk
     9840  01-01-1980 00:00   splits/base-el.apk
     9550  01-01-1980 00:00   splits/base-gl.apk
    10148  01-01-1980 00:00   splits/base-ml.apk
     9545  01-01-1980 00:00   splits/base-nl.apk
     9540  01-01-1980 00:00   splits/base-pl.apk
     9561  01-01-1980 00:00   splits/base-sl.apk
     9565  01-01-1980 00:00   splits/base-tl.apk
     9707  01-01-1980 00:00   splits/base-am.apk
     9877  01-01-1980 00:00   splits/base-km.apk
    10026  01-01-1980 00:00   splits/base-bn.apk
     9502  01-01-1980 00:00   splits/base-in.apk
    10045  01-01-1980 00:00   splits/base-kn.apk
     9693  01-01-1980 00:00   splits/base-mn.apk
     9511  01-01-1980 00:00   splits/base-ko.apk
     9700  01-01-1980 00:00   splits/base-lo.apk
     9592  01-01-1980 00:00   splits/base-ro.apk
     9540  01-01-1980 00:00   splits/base-sq.apk
     9648  01-01-1980 00:00   splits/base-ar.apk
    10469  01-01-1980 00:00   splits/base-fr.apk
     9544  01-01-1980 00:00   splits/base-hr.apk
    10008  01-01-1980 00:00   splits/base-or.apk
     9901  01-01-1980 00:00   splits/base-mr.apk
    10884  01-01-1980 00:00   splits/base-sr.apk
     9738  01-01-1980 00:00   splits/base-ur.apk
     9502  01-01-1980 00:00   splits/base-tr.apk
    10118  01-01-1980 00:00   splits/base-as.apk
     9573  01-01-1980 00:00   splits/base-bs.apk
     9529  01-01-1980 00:00   splits/base-cs.apk
    10605  01-01-1980 00:00   splits/base-es.apk
     9486  01-01-1980 00:00   splits/base-is.apk
     9502  01-01-1980 00:00   splits/base-ms.apk
     9574  01-01-1980 00:00   splits/base-et.apk
  2445880  01-01-1980 00:00   splits/base-master.apk
     9536  01-01-1980 00:00   splits/base-it.apk
     9657  01-01-1980 00:00   splits/base-lt.apk
    11333  01-01-1980 00:00   splits/base-pt.apk
     9557  01-01-1980 00:00   splits/base-eu.apk
     9852  01-01-1980 00:00   splits/base-gu.apk
     9616  01-01-1980 00:00   splits/base-hu.apk
     9737  01-01-1980 00:00   splits/base-ru.apk
     9759  01-01-1980 00:00   splits/base-lv.apk
     9504  01-01-1980 00:00   splits/base-zu.apk
     9517  01-01-1980 00:00   splits/base-sv.apk
     9509  01-01-1980 00:00   splits/base-sw.apk
     9586  01-01-1980 00:00   splits/base-iw.apk
     9682  01-01-1980 00:00   splits/base-hy.apk
     9693  01-01-1980 00:00   splits/base-ky.apk
    10173  01-01-1980 00:00   splits/base-my.apk
     9566  01-01-1980 00:00   splits/base-az.apk
     9507  01-01-1980 00:00   splits/base-uz.apk
    21821  01-01-1980 00:00   splits/base-en.apk
    11565  01-01-1980 00:00   splits/base-zh.apk
     6357  01-01-1980 00:00   toc.pb
---------                     -------
 21720880                     98 files
```

指定设备apks，内部的split apk会少很多，大量无关语言包，密度都没有了
```
aven-mac-pro-2:debug aven$ unzip -l emulator.apks 
Archive:  emulator.apks
  Length      Date    Time    Name
---------  ---------- -----   ----
     6484  01-01-1980 00:00   splits/dynamicfeature-xxhdpi.apk
    16223  01-01-1980 00:00   splits/dynamicfeature-master.apk
    69091  01-01-1980 00:00   splits/base-xxhdpi.apk
    21821  01-01-1980 00:00   splits/base-en.apk
  2445880  01-01-1980 00:00   splits/base-master.apk
      411  01-01-1980 00:00   toc.pb
---------                     -------
  2559910                     6 files
```

## 生成的apks分类情况
如果只有一个基础库，那么所有产生的apk名字前缀都是base
base-master.apk： Contains the code and resources for the base module
base-armeabi_v7a.apk, base-arm64_v8a.apk, etc.：ABI configuration splits
base-xhdpi.apk, base-mdpi.apk, etc. ： Screen density configuration splits
base-ko.apk, base-fr.apk, etc. ： Language configuration splits

## 创建动态模块
1. Create New Module, Dynamic Feature Module
2. Configure your new module: 选择base库，配置Module Name，配置包名，配置API Level
3. Configure On-Demand Options: Module title（用户可见）；Enable On-Demand; Enable Fusing

## 创建动态模块生成的配置

模块AndroidManifest.xml，其中title配置待base中，需要在模块下载器提示用户时使用

```xml
<dist:module
   dist:onDemand="true"
   dist:title="@string/title_payments">
   <dist:fusing include="true" />
</dist:module>
```

* 模块的build.gradle

```
apply plugin: 'com.android.dynamic-feature'
```

* base的build.gradle，在android下添加配置

```
dynamicFeatures = [":payments"]
```

* 在模块的build.gradle添加base依赖

```
dependencies {
   implementation project(':app')
}
```

## Module manifest配置

参考文档： https://developer.android.google.cn/studio/projects/dynamic-delivery#dynamic_feature_manifest


* 声明dist命名空间

```
xmlns:dist="http://schemas.android.com/apk/distribution
bundle打包自动配置属性，无需开发者配置
split="split_name"
android:isFeatureSplit="true | false">
```
* module节点配置

```
instant配置：dist:instant="true | false"
模块名配置：dist:title="@string/feature_name"
4.4及以下支持：<dist:fusing dist:include="true | false" />
```

* delivery配置

```
按需加载：<dist:on-demand/>
安装加载：<dist:install-time/>
```

## 参考
* https://developer.android.com/guide/app-bundle/ 
* https://codelabs.developers.google.com/codelabs/your-first-dynamic-app/index.html#6
* https://developer.android.google.cn/studio/projects/dynamic-delivery#dynamic_feature_manifest


