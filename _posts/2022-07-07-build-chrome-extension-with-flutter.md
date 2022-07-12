---
layout: post
title: "真香！我用Flutter开发了一款Chrome插件工具"
description: ""
header_image: assets/img/2022-07-07-01.jpeg
keywords: ""
tags: [Flutter,Chrome]
---
{% include JB/setup %}

![img](/assets/img/2022-07-07-01.jpeg)


* 目录

{:toc #markdown-toc}

## 背景

Chrome插件也称作Chrome Extension，是运行在Chrome浏览器上的插件/应用。最开始插件和应用是两个细分方向，2018 Chrome Apps彻底退役了，只保留了Chrome Extension，2022年了，插件市场仍然健在😁。“上一次开发Chrome插件，还是上一次”，用的还是传统JS开发，没有想到有一天会有一门“新的”语言“再作冯妇”。


Chrome插件"寄生"在浏览器上，可嵌入网页执行js，可后台执行js。通过开发插件，可以基于浏览器做一些个性化的增强功能，比如JSON解析的工具，二维码的工具。也可以做一些针对特定网页的工具，这种情况常见于你想提取一个网页的内容做二次加工，但是你不是网站的维护者，或者不适合从网站本身做修改。
﻿

这个时候自己动手做一个插件就非常合适了。本文记录了笔者将Flutter开发的小工具迁移到Chrome的故事。

## 准备工作

首先你得会Flutter开发，Flutter的环境是必要的。
﻿
截止2022.07，稳定版本的Flutter已经到了3.0.4，但是你用Flutter 2也是可以的，最低要求是支持Flutter Web。
然后是Chrome浏览器了，本文的插件是基于Manifest V3的，Chrome版本太老的话可能仅支持Manifest V2，建议升级上来，因为V3在22年下半年就是Chrome Extension开发是必须使用的版本，用V2开发早晚也得迁移到V3。

下面是一些环境参考：

|环境   |参考  |
| ---| ---|
|Flutter配置 |https://docs.flutter.dev/get-started/install﻿ |
|Chrome浏览器|https://www.google.com/chrome/﻿|
|Chrome插件 | https://developer.chrome.com/docs/extensions/﻿|

安装完成后，用flutter doctor检查一下，flutter可以跨平台开发，不需要所有链路都满足，保证flutter web的支线就行：

```
yz@B-J58KMD6R-1923 app_bundle_diff % flutter doctor  
Doctor summary (to see all details, run flutter doctor -v):
[✓] Flutter (Channel stable, 3.0.4, on macOS 11.2.3 20D91 darwin-x64, locale zh-Hans-CN)
[✓] Android toolchain - develop for Android devices (Android SDK version 32.0.0)
[!] Xcode - develop for iOS and macOS (Xcode 12.5.1)
    ✗ Flutter requires Xcode 13 or higher.
      Download the latest version or update via the Mac App Store.
    ✗ CocoaPods not installed.
        CocoaPods is used to retrieve the iOS and macOS platform side's plugin code that responds to your plugin usage on the Dart side.
        Without CocoaPods, plugins will not work on iOS or macOS.
        For more info, see https://flutter.dev/platform-plugins
      To install see https://guides.cocoapods.org/using/getting-started.html#installation for instructions.
[✓] Chrome - develop for the web
[✓] Android Studio (version 2021.2)
[✓] Android Studio (version 3.6)
[✗] Cannot determine if IntelliJ is installed
    ✗ Directory listing failed
[✓] Connected device (2 available)
[✓] HTTP Host Availability
​
! Doctor found issues in 2 categories.
```

## 插件开发

接下来就是开发功能了，提前构思好你想要做的事情。如果没想好，也可以先啥都不改，直接用模板项目自动生成的计数器作为你的插件内容。笔者这里开发了几个功能：

### 需求功能

*  提取伙伴迭代的基线清单，按大小顺序排序，支持静转动过滤。
*  解析AndroidX官方的ClassMapping表单
*  其他存储日志的解析
*  二维码生成，这个是拿来主义的，集成进来

目前实现的效果大概长这样

![img](/assets/images/extension-preview.gif)

### 基线清单

伙伴上某一个具体迭代或者安装包都是可以看到他的基线清单的，满足普通查阅诉求。笔者经常需要过滤基线，看大小，所有静转动。没准哪天还会想按更新时间做聚合，看看那些模块常年不更新。
﻿
动手开干。第一步绑定迭代，基于迭代的id，我们可以获取迭代基线。有了基线就可以做更多的聚合/过滤了。
第一次进入“基线清单”，会提示开发者是否要绑定一下迭代，将迭代地址输入到绑定页面后会自动解析，然后同步基线。
﻿
![img](/assets/images/bind-project-id.gif)

下一次再次进行默认会加载已经绑定的基线，如果需要切换基线通过右下角菜单重新绑定即可。基线url不可以乱输，会做简单的校验，

![img](/assets/images/rebind-project-id.gif)

如果是Android开发，你可能知道portal里面也有一个基线同步的功能，大致思路是类似的，都是通过基线id做的同步。只不过我这里简化了，不需要你自己去找id，直接走url里面提取了id。

保存迭代的id，用的是LocalStorage，可以通过浏览器的开发者工具查看保存的内容。

![img](/assets/images/project-id.gif)

这里面涉及的内容其实并不多，主要是列表，选择器，网络请求（注意跨域问题）。如果直接用js去开发（或者套用React/Vue），“半桶水”的情况头都要大了。利用Flutter，可以站在巨人的肩上，不但轻车熟路，不喜欢了，推到修改也容易。
﻿
下面是拉基线的核心代码

```dart
void refreshProject() {
  context.read<DataProvider>().getString("projectUniqueId").then((value) {
    debugPrint("local projectUniqueId=$value");
    if (value != null && value.isNotEmpty) {
      return loadProjectByID(value, context);
    } else {
      return Future.value("");
    }
  }).then((value) {
    if (value.isNotEmpty) {
      return parseDependencyAsync(value);
    } else {
      return Future.value(null);
    }
  }).then((value) {
    if (value != null) {
      setState(() => bundleList = value.items);
      if (value.projectName != null) {
        context.showSnackBar("基线加载完成, ${value.projectName}");
      } else {
        context.showSnackBar("基线加载完成");
      }
    } else {
      var snackBar = SnackBar(
        content: const Text("需要先设置基线，前往设置？"),
        action: SnackBarAction(
          label: "好的",
          onPressed: () {
            openSettingPage(context);
          },
        ),
      );
      ScaffoldMessenger.of(context).showSnackBar(snackBar);
    }
  }).onError((error, stackTrace) {
    context.showSnackBar("基线加载失败");
    debugPrintStack(stackTrace: stackTrace);
  });
}
```

### Class映射

在做AndroidX升级的时候，时常需要查询Support中Class对应的AndroidX中Class的的对应关系，虽然有规律，但是成百上千，要记住并不容易。因此有一个查询的方法该多好。过去每次都得找到官网，再去搜索，网络还比较慢。

![img](/assets/images/class-mapping.gif)

除了可以自己检索Class映射，也可以做定制格式的输出，比如导出为properties配置文件。


```dart
void refreshList({bool foreRefresh = false}) {
  parseClassMappingCSVFile(selectedFile!, useCustom).then((data) {
    if (data == null || data.isEmpty) {
      showError();
    } else {
      setState(() {
        fileItems = data.entries.toList();
      });
      showSuccess();
    }
  }).onError((error, stackTrace) {
    showError();
  });
}
```

其他的功能模块就不介绍了，读者可以去实现你自己的插件内容。

## 插件产物

到这里我们一直是在开发Flutter，和Chrome Extension没有半毛钱关系。所以怎么要才可以让Flutter开发的App变成Chrome插件呢？

### crx 安装包

首先需要认识一些Chrome Extension的安装包。好比Android手机安装应用需要APK安装包，Chrome浏览器安装插件则需要CRX。
你肯能猜到了CRX也是一个ZIP压缩包，里面包含了HTML/CSS/JS，图片和其他资源，运行在一个独立的沙箱环境，可以通过API和浏览器进行交互。

> Extensions are zipped bundles of HTML, CSS, JavaScript, images, and other files used in the web platform. Extensions can modify web content that users see and interact with. They can also extend and change the behavior of the browser itself.

平时开发/调试，通过浏览器的开发者模式，直接加载插件项目，以“已解压的插件扩展程序”来运行。上架前打包为ZIP提交Chrome Web Store审核，审核通过后可以发布。

https://developer.chrome.com/docs/extensions/mv3/architecture-overview/﻿

```
yz@mbp home % unzip -l release/extension-0.1.0.zip 
Archive:  release/extension-0.1.0.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  07-06-2022 19:20   assets/
      217  07-06-2022 19:20   assets/AssetManifest.json
   880022  07-06-2022 19:20   assets/NOTICES
      208  07-06-2022 19:20   assets/FontManifest.json
        0  07-06-2022 19:20   assets/packages/
        0  07-06-2022 19:20   assets/packages/cupertino_icons/
        0  07-06-2022 19:20   assets/packages/cupertino_icons/assets/
   283452  01-26-2022 15:05   assets/packages/cupertino_icons/assets/CupertinoIcons.ttf
        0  07-06-2022 19:20   assets/fonts/
  1614500  03-25-2022 21:45   assets/fonts/MaterialIcons-Regular.otf
        0  07-06-2022 19:20   assets/data/
     8820  07-06-2022 12:01   assets/data/huoban.png
   197059  06-07-2022 11:35   assets/data/androidx-class-mapping.csv
     7166  07-06-2022 19:20   flutter_service_worker.js
    69215  07-06-2022 16:22   icon.png
    10028  07-06-2022 16:24   icon128.png
     1581  07-06-2022 16:25   icon16.png
     4026  07-06-2022 16:25   icon48.png
        0  07-06-2022 19:20   icons/
    43264  07-06-2022 16:27   icons/Icon-192.png
    43264  07-06-2022 16:27   icons/Icon-maskable-192.png
    43264  07-06-2022 16:27   icons/Icon-maskable-512.png
    43264  07-06-2022 16:27   icons/Icon-512.png
     1313  07-06-2022 19:20   index.html
  1889642  07-06-2022 19:20   main.dart.js
      525  07-06-2022 16:50   manifest.json
       96  07-06-2022 19:20   version.json
---------                     -------
  5140926                     27 files
```

下面介绍一些插件中开发经常涉及的内容：

|内容|说明|
|---|---|
|manifest.json|【必要】插件的配置清单|
|icon|【必要】插件的图标，路径不限制，一般根目录就行|
|Toolbar icon|【必要】插件在浏览器工具栏上的卡槽，一般是个按钮，会有弹出框|
|Service worker|【非必要】后台服务，一般用于事件监听。|
|Content scripts|【非必要】内容脚本，非必要。这个js是注入到当前网页中执行的，可以读写页面DOM|
|UI elements|【非必要】其他的交互场景下插件的露出，非必要。比如右键菜单，通知，搜索框，快捷键|

不同场景下的js之间是可以通信的，利用message机制

![img](/assets/images/extension-messeger.png)

### Manifest V3

crx里面一定要有manifest.json，类似AndroidManifest.xml，配置了应用的各种组件，属性，权限声明。

```json
{
  "name": "My Extension",
  "description": "A nice little demo extension.",
  "version": "2.1",
  "manifest_version": 3,
  "icons": {
    "16": "icon_16.png",
    "48": "icon_48.png",
    "128": "icon_128.png"
  },
  "background": {
    "service_worker": "background.js"
  },
  "permissions": ["storage"],
  "host_permissions": ["*://*.example.com/*"],
  "action": {
    "default_icon": "icon_16.png",
    "default_popup": "popup.html"
  }
}
```

manifest目前主要有两种V2和V3，建议新开发的使用V3，两个版本的配置和约束都有差异。
﻿https://developer.chrome.com/docs/extensions/mv3/intro/mv3-overview/

### 产物打包

可以自己打包，自己维护密钥。现在也有可以直接压缩zip然后交给Chrome Web Store打包。
我们已经知道了CRX就是一个待manifest.json的资源包，因此在先把flutter打包成js产物，然后再打包为zip就可以了。

```shell
#!/usr/bin/env bash
##################################################################
##
##  Build flutter web for Chrome extension
##
##
##  Author: Chaobin Wu
##  Email : chaobinwu89@gmail.com
##
#################################################################
echo "remove build/"
rm -rf build/web
echo "build web assets"
flutter build web --web-renderer html --csp
echo "zip as .crx"
cd build/web && zip -r extension.zip *
echo "remove invalid meta files"
zip -d extension.zip "__MACOSX*"
zip -d extension.zip "*/.DS_Store"
echo "copy to build/"
mv extension.zip ../extension.zip
unzip -l ../extension.zip
```
![img](/assets/images/extension-packager.png)

![img](/assets/images/packager-alert.png)

## 插件上架

上架主要是要准备好物料，比如logo，宣传图，介绍文字，尺寸都有要求。
﻿https://developer.chrome.com/docs/webstore/publish/


|物料|要求|
|----|----|
|Title from package|从manifest的name自动提取的，不可以在送审后台修改|
|Summary from package|从manifest的description自动提取的，不可以在送审后台修改|
|Version|从manifest的version自动提取的，不可以在送审后台修改|
|Description|后台填写，不可以和“Summary from package”重复，不可以有歧义，可以写中文|
|Store icon|应用图标，128*128|
|Screenshots|宣传图（轮播图），1280*800，或者640*400，JPG或者PNG，不能带透明度|
|Single purpose|应用目的描述，可以一句话秒速清楚是插件是干什么的|
|Permission justification|权限使用说明，如果用了权限，这里要说明为什么|

﻿
整个审批比前些年更加规范了（光看表单内容可见一斑）。不过审批速度仍然很快，一般半天时间审批就能完成。

|表单|审核|
|----|----|
|![img](/assets/images/web-store-preview.png) | ![img](/assets/images/web-store-upload.png)|



如果审批被拒了，会有邮件通知，告知你违反的条款，根据要求重新调整后，可以重新提交。如果通过了好像不会通知，需要自己去后台查看。

## 小结

Flutter除了做移动端开发，已然全平台制霸，仿佛一套代码走天下已经来临，赶快上车。

