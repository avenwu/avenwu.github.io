---
layout: post
title: "打造你的专属Mac工具"
description: ""
header_image: /assets/images/2019-04-22-01.png
keywords: ""
tags: [Mac]
---
{% include JB/setup %}
![2019-04-22-01.png](/assets/images/2019-04-22-01.png)

## 引言
作为开发者，经常会使用各种工具来简化一些工作，比如好玩的Chrome插件，IDE插件，Web在线服务等等，有些可能没有简单的可视化展现，比如大量的`*nix`命令。无论形式是哪种，目标是都是类似的。

笔者在日常工作中，也会有这样、那样的诉求，文本介绍一下如何开发一款Mac助手程序，来更好满足一些个性化的需求。

本文涵盖了以下知识点：
* Android Debug Bridge采集设备信息&App信息
* Node如何开启新的终端窗口并执行脚本
* JS解决CPU密集型任务&UI卡顿
* 使用Node调用Golang服务/工具
* 了解mac下的安装包构成
* 制作dmg应用安装包
* 尝试开发你的mac应用

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

由于我们的处理毕竟是有限的，如果需要快速打开一个终端，执行预设的脚本怎么操作呢？是不是手工切换面板，打开Terminal，输入命令？

## 启动新的Terminal窗口并执行任务

记得在分析ReactNative的时候，知道RN在最后一步从一个终端打开了另一个终端，并且自动执行的脚本任务。

现在我们也需要在App内，当用户点击任务时，自动给他打开一个终端，为了实现这个效果，我们先回过头，去看看如何实现。

在之前的文章中，我们已经分析到了，启动新的终端是通过执行`startServerInNewWindow`方法，详细参考 [React Native启动流程#run-android流程](http://blog.hacktons.cn/2018/04/28/react-native-startup/#run-android%E6%B5%81%E7%A8%8B)

总结一下，就是根据不同平台差异化处理：
>在macOS上，是通过`open`命令，由`-a`参数指定我们的`Terminal`终端程序，第二个参数是被执行脚本。
接下来看下RN的`launchPackager.command`是如何实现的:

**[launchPackager.command](https://github.com/facebook/react-native/blob/master/scripts/launchPackager.command)**

```javascript
#!/bin/bash
# Copyright (c) Facebook, Inc. and its affiliates.
#
# This source code is licensed under the MIT license found in the
# LICENSE file in the root directory of this source tree.

# Set terminal title
echo -en "\\033]0;Metro Bundler\\a"
clear

THIS_DIR=$(cd -P "$(dirname "$(readlink "${BASH_SOURCE[0]}" || echo "${BASH_SOURCE[0]}")")" && pwd)

# shellcheck source=/dev/null
. "$THIS_DIR/packager.sh"

if [[ -z "$CI" ]]; then
  echo "Process terminated. Press <enter> to close the window"
  read -r
fi
```
现在我们可以提取核心逻辑，实现我们的需求

```shell
open -a /Applications/Utilities/Terminal.app launch.command
```

## 安装包检测

关于这一块，主要是通过静态检测安装包，来发现一些问题，比如类的检测，文件检测。
还有就是检测是否存在的大图片和潜在的内存泄漏图片，这一块目前还处于实验性质暂时不多做介绍。

安装包的实际上是集成了笔者之前开发的一个Golang脚本。图片检测比较简单，是直接用JS写的，可以按照一定规则拆包解析APK内的所有图片资源，并根据参数设置，返回超出阈值的图片列表。
相比`cli`形式，我们可以做一些UI上的处理，比如分类展示，目标图片查看下。面是检测后的效果：

![next-big-image-scan.png](/assets/images/next-big-image-scan.png)

下面讲一下，怎么集成现成的服务到App内，从而避免冗余开发：
利用`child_process#execFile`，把我们的脚本内置到工程中，调用的时候当初可执行文件直接使用，然后根据脚本的要求传入参数，解析输出结果即可。

**下面是部分源码：**

```javascript
execFile(binPath, args, { cwd: os.tmpdir() }, (error, stdout, stderr) => {
    com.setState({
        loading: false,
        operation_state: STATE_COMPLETE_SUCCESS
    })
    if (error) {
        console.error(`${error}`);
        com.updateConsole(`${error}`);
        com.setState({
            snackbar: {
                open: true,
                variant: "error",
                message: "安装检测失败，请查看日志",
            }
        })
        return;
    }
    stdout && com.updateConsole(`安装检测完成，请查看日志:\n${stdout}`);
    stderr && com.updateConsole(`安装检测失败，请查看日志:\n${stderr}`);
    electronFs.readFile(outputPath, (err, data) => {
        if (err) {
            com.updateConsole(`exec error: ${err}`);
            com.setState({
                snackbar: {
                    open: true,
                    variant: "error",
                    message: "读取报告失败，请查看日志",
                }
            })
            return;
        }
        com.setState({
            scan_result: JSON.parse(new String(data))
        })
    });
    com.setState({
        snackbar: {
            open: true,
            variant: "success",
            message: "安装检测完成，请查看日志",
        }
    })
});
```

在开发无界面批处理脚本时，往往是串行单线程的，现在由于我们的前端界面使用的是js开发，因此在解析图片过程中有大量的循环，递归操作，属于CPU要求比较高的任务，为了避免UI卡顿(精度条停滞)，需要简单处理下，通过开辟专用的图片处理线程解决卡顿问题。

* 在浏览器内，可以通过[Web Workers](https://developer.mozilla.org/zh-CN/docs/Web/API/Worker)技术实现;
* 在Nodejs内，可以通过[Worker Thread](https://nodejs.org/api/worker_threads.html)技术实现，可以理解为Node下的`Web Workers`技术，API基本一致，需要在`Node 12`开始默认支持；
* Nodejs内，还可以通过[child_process](https://nodejs.org/api/child_process.html)子进程实现，也很方便。
* 在Electron内，根据API说明，和实际测试，`Web Workers`和`child_process`都可以实现， [Multithreading](https://electronjs.org/docs/tutorial/multithreading)

为了支持Web Workers多线程，根据官方说明，需要开启设置：
```javascript
let win = new BrowserWindow({
  webPreferences: {
    nodeIntegrationInWorker: true
  }
})
```

**`child_process`实现版本**

```javascript
// 子进程，接收消息，执行耗时操作，并发送结果
process.on('message', params => {
    console.log('receive message on subprocess', params)
    findLargeImage(params.dir, params.config).then(images => {
        console.log('scan complete', images);
        process.send(images)
    })
})

// 父进程，实例化子进程，接收回调消息
unzipFile(src, dir).then(() => {
  console.log('unzip done');
  const scriptsDir = path.resolve(__dirname);
  const backgroundImageScanJs = path.resolve(scriptsDir, 'backgroundImageScan.js');
  const processor = fork(backgroundImageScanJs);
  processor.send({ dir: dir, config: { oomBytes: 5 * 1024 * 1024, oomDpi: 480 } });
  processor.on('message', data => {
      console.log(data);
      com.setState({
          operation_state: STATE_COMPLETE_SUCCESS,
          loading: false,
          data: data,
      });
  })
}).catch(err => {
  com.setState({
      operation_state: STATE_COMPLETE_FAILED,
      loading: false,
      snackbar: {
          open: true,
          variant: 'error',
          message: '解压失败，请检查装包:' + src,
      },
  });
})
```

**`Web Workers`实现版本**
```javascript
// 子线程，接收消息，执行耗时操作，并发送结果
onmessage = e => {
    const params = e.data;
    console.log('receive message from main thread', params)
    findLargeImage(params.dir, params.config).then(images => {
        console.log('scan complete');
        postMessage(images)
    })
}
// 调用线程，实例化子线程，接收回调消息
unzipFile(src, dir).then(() => {
  console.log('unzip done');
  const scriptsDir = path.resolve(__dirname);
  const backgroundImageScanJs = path.resolve(scriptsDir, 'backgroundImageScan.js');
  const worker = new Worker(backgroundImageScanJs)
  worker.postMessage({ dir: dir, config: { oomBytes: 5 * 1024 * 1024, oomDpi: 480 } })
  worker.onmessage = e => {
      const data = e.data;
      console.log('receive work result', data)
      com.setState({
          operation_state: STATE_COMPLETE_SUCCESS,
          loading: false,
          data: data,
      });
  }
}).catch(err => {
  com.setState({
      operation_state: STATE_COMPLETE_FAILED,
      loading: false,
      snackbar: {
          open: true,
          variant: 'error',
          message: '解压失败，请检查装包:' + src,
      },
  });
})
```

## 仓库Diff预览

这个功能，其实存粹是一个中间产物。项目开发中，存在几十个的仓库，每次版本同步都是一个费力活，这个模块的作用是快速查看一个仓库的版本之间是否需要合并，也就是有没有diff。

![next-repo-diff.png](/assets/images/next-repo-diff.png)

当然现在这个模块的作用已经不是很大，因为在合并代码流程上做了改进。

## Dmg发行包制作
在Mac下安装软件有几种不同方式，通过dmg镜像拖拽安装是比较常见的一种。下面我们讲一下怎么制作镜像包。

制作dmg的方法会有很多，既可以用mac下的自盘管理工具，也可以通过appdmg等node程序；简单起见，我们使用的是`electron-forge`，这个工具最终调用了appdmg，只不过经过了几次封装，可以通过一些json配置来生成安装包。

先看几个dmg安装包的示例效果

![wps dmg](/assets/images/wps-dmg-installer.png)

![qq dmg](/assets/images/qq-dmg-installer.png)

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

下面是WPS和QQ的磁盘图标，利用PS，我们也仿照着做一个图标，简单加上58的水印：

![dmg logo.png](/assets/images/dmg-logo-compare.png)

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
这个库最终依赖的就是[https://github.com/LinusU/node-appdmg](https://github.com/LinusU/node-appdmg)。

### 发布&安装应用
标准发布时需要将App签名，然后提交到AppStore内，这个需要Apple账号，这里我们是本地使用，不需要上传市场，因此只需要上传到服务器提供下载就行。

![mac-app-list.png](/assets/images/mac-app-list.png)

## 参考资料

* [https://electronjs.org/docs](https://electronjs.org/docs)
* [https://nodejs.org/api/synopsis.html](https://nodejs.org/api/synopsis.html)
* [http://photonkit.com/components/](http://photonkit.com/components/)
* [https://zaiste.net/nodejs-child-process-spawn-exec-fork-async-await/](https://zaiste.net/nodejs-child-process-spawn-exec-fork-async-await/)
* [https://github.com/facebook/react-native/blob/master/scripts/launchPackager.command](https://github.com/facebook/react-native/blob/master/scripts/launchPackager.command)
