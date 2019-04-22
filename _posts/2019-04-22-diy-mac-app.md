---
layout: post
title: "打造你的专属Mac工具"
description: ""
header_image: /assets/images/2019-04-22-01.png
keywords: ""
tags: []
---
{% include JB/setup %}
![2019-04-22-01.png](/assets/images/2019-04-22-01.png)

## 引言
作为开发者，经常会使用各种工具来简化一些工作，比如好玩的Chrome插件，IDE插件，Web在线服务等等，有些可能没有简单的可视化展现，比如大量的`*nix`命令。无论形式是哪种，目标是都是类似的。

笔者在日常工作中，也会有这样、那样的诉求，文本介绍一下如何开发一款Mac助手程序，来更好满足一些个性化的需求。

![next-install-sample.png](/assets/images/next-install-sample.png)

## 需求收集

>开发之前首先明确想要什么？

我这里其实比较简单，就是开发一个聚合助手软件，围绕解决以下几个痛点：
1. 平常用的频率高，但是不方便的工具；
2. 操作简单，但是效果不直观的；
3. 频率高，操作简单，但是使用时不容易记忆的；

综合考虑实现成本，目前开发的功能模块如下：

![Next-for-Mac.png](/assets/images/Next-for-Mac.png)

1. Android设备信息展示
2. 应用信息检索
3. APK图片检测
4. 仓库Diff预览

![next-url-decode](/assets/images/next-url-decode.png)

## 设备信息
设备信息有很多，只采集一些常用的，比如排查时经常问的设备型号，厂商，系统版本，分辨率等。

**硬件信息**

硬件信息的采集主要依靠ADB获取，比如:
* `adb shell dumpsys battery`
* `adb shell wm size`
* `adb shell wm density`

**系统信息**

系统版本信息类似，比如：
* `adb shell getprop ro.build.version.release`
* `adb shell getprop ro.product.brand`

**应用列表**

获取应用列表，通过`pm`服务，可以拿到系统应用和三方应用列表
* `adb shell pm list packages -3`
* `adb shell pm list packages -s`


效果类似如下：

![next-device-info.png](/assets/images/next-device-info.png)

## 应用信息检索

查看设备上某一App的信息，主要用于辅助逆向分析。

> adb shell dumpsys package [pakcage name]

应用信息比较有用的除了版本号之外，还有调试状态，权限清单等

另外一个经常使用的是查看当前顶层页面的Activity，这个信息可以快速告诉我们当前页面的实现类，是什么载体，那个业务线维护等等。

## APK图片检测
// TODO

## 仓库Diff预览
// TODO

## DMG发行包制作
在Mac下安装软件有几种不同方式，通过dmg镜像拖拽安装是比较常见的一种。下面我们讲一下怎么制作镜像包。

制作dmg的方法会有很多，既可以用mac下的自盘管理工具，也可以通过appdmg等node程序；简单起见，我们使用的是`electron-forge`，这个工具最终调用了appdmg，只不过经过了几次封装，可以通过一些json配置来生成安装包。

### 物料准备&配置
安装包涉及的UI，这里有三个，一个是App的LOGO，一个是安装包的背景图，还有一个是磁盘镜像图标。

默认情况下不需要配置任何上图里面的参数，就可以打出一个dmg，但是如果我们需要修改的话，就需要知道如何替换。

**磁盘图标**

镜像文件的图标的图标和应用图标很容易搞错

```json
"electronInstallerDMG": {
	"icon": "assets/volume.icns"
}
```
![next-dmg-image.png](/assets/images/next-dmg-image.png)

**配置背景图**

安装界面的背景图通过`background`来配置
```json
"electronInstallerDMG": {
	"background": "assets/background.tiff"
}
```

>background需要固定尺寸，如果适配的话可以做一份@2x图，并打包成tiff格式

```shell
tiffutil -cathidpicheck assets/background.png assets/background@2x.png -out assets/background.tiff
```

**设置app图标**

应用图标需要在`electronPackagerConfig`节点配置一个`icon`,也需要是icns格式；icns格式这个网上很多转换工具；

```json
"electronPackagerConfig": {
	"packageManager": "yarn",
	"icon": "assets/icon.icns"
}
```

### 生成安装包

现在，看一下如果打包一个模板样例工程。通过模板工具创建electron工程，只需要简单几个命令：

```shell
npm install -g electron-forge
electron-forge init my-new-project
cd my-new-project
electron-forge start
```

如此你便跑起了最简单的一个项目。在创建好的electron项目默认会有以下配置`package.json`：

```json
{
  "make_targets": {
    "win32": [
      "squirrel"
    ],
    "darwin": [
      "zip"
    ],
    "linux": [
      "deb",
      "rpm"
    ]
  },
  "electronPackagerConfig": {},
  "electronWinstallerConfig": {
    "name": ""
  },
  "electronInstallerDebian": {},
  "electronInstallerRedhat": {},
  "github_repository": {
    "owner": "",
    "name": ""
  },
  "windowsStoreConfig": {
    "packageName": ""
  }
}
```

**配置dmg输出文件**

>通过`electron-forge make`可以编译出安装包，但是你会发现`out`目录下并没有dmg文件；

结合前面介绍的图标配置，为了得到dmg文件，我们需要修改下配置：

```json
"electronInstallerDMG": {
  	"background": "path/to/image.png",
  	"icon": "path/to/icon.icns",
  	"format": "ULFO"
}
```
这个配置实际上也只是包裹了一下，因此想明确知道含义，需要看文档了；

[electron-installer-dmg](https://github.com/electron-userland/electron-installer-dmg#api)

截取几个字段说明：

```
background - String
Path to the background for the DMG window. Background image should be of size 658 × 498.

icon - String
Path to the icon to use for the app in the DMG window.
```
这个库最终依赖的就是[appdmg](https://github.com/LinusU/node-appdmg), 因此我们可以得到一个更好的示意说明：

![dmg](/assets/images/appdmg-help.png)

### 发布&安装应用
标准发布时需要将App签名，然后提交到AppStore内，这个需要Apple账号，这里我们是本地使用，不需要上传市场，因此只需要上传到服务器提供下载就行。

![mac-app-list.png](/assets/images/mac-app-list.png)