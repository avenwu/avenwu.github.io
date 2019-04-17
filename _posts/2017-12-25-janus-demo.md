---
layout: post
title: "Janus 二元神漏洞测试"
description: ""
header_image: /assets/img/2017-12-25-01.jpg
keywords: "Janus"
tags: [Janus]
---
{% include JB/setup %}
![img](/assets/img/2017-12-25-01.jpg)

## 背景

12月9号，Andorid对外曝光了一个名为`Janus`的重量级系统漏洞`CVE-2017-13156)`, 由安全研究公司[Guard Square](https://www.guardsquare.com/en/blog/new-android-vulnerability-allows-attackers-modify-apps-without-affecting-their-signatures)发现。
`Janus`原意是神话中的二元身，用于描述这个漏洞还真是贴切。

![Apk_Dex_Dual](/assets/img/Apk_Dex_Dual.png)

整个漏洞其实建立在文件校验规则之上：

>
一个文件即是APK，又是DEX，在安装APK和执行阶段的校验规则差异，导致可以在APK头部附加一个恶意DEX来欺骗系统

下面我们从市场上随意下载一个apk来做测试。

## 测试APK

`本文涉及的测试APK，只用于单击研究之用，请勿恶意散播或上传，由此引发的纠纷与作者无关`

可以从豌豆荚，应用宝等市场下一个测试用的APK,为了方便，我们需要选用一些体积较小的apk，如果apk较大很有可能经过了分包，替换工作会麻烦点。
![快看漫画](/assets/img/测试apk.png)

这里比较恶心的是，下载到的apk并不是我们选取的安装包，而是豌豆荚市场，既然豌豆荚这么强势要入镜，那么姑且直接分析豌豆荚市场吧。

![快看漫画](/assets/img/豌豆荚apk.png)

```
MD5 (Wandoujia_224660_web_inner_referral_binded.apk) = d3c1d9b2a74a3f8fd9fce38d38423c58
```

## 签名检查

首先检查下豌豆荚的这个apk是不是v2签名的，因为我们要测试的`Janus`只能在v1下验证

检查签名信息可以通过*.SF来确认，根据公开信息，如果v2签名的话，会在SF文件内写入一个字段` X-Android-APK-Signed:2`；
`豌豆荚`的SF文件名字是`META-INF/DEAMON2.SF`, 比较幸运啊，可以确认其使用的就是v1签名。

```
aven$ unzip -l Wandoujia_224660_web_inner_referral_binded.apk |grep META-INF
   120009  12-15-17 14:38   META-INF/MANIFEST.MF
   120130  12-15-17 14:38   META-INF/DEAMON2.SF
      891  12-15-17 14:38   META-INF/DEAMON2.RSA
```

```
aven$ unzip -p Wandoujia_224660_web_inner_referral_binded.apk META-INF/DEAMON2.SF|less
Signature-Version: 1.0
SHA1-Digest-Manifest-Main-Attributes: jq/6qzaCk3O+H4OBJsDhMXm+FvE=
Created-By: 1.6.0_30 (Sun Microsystems Inc.)
SHA1-Digest-Manifest: Dts4zfEM9pZstNDahVfVh4e4jGA=

Name: res/drawable-xhdpi-v4/il.png
SHA1-Digest: QCves3Cr/wm3X2w4PR4ESXGMBOw=

Name: res/layout/dh.xml
SHA1-Digest: DCuKb0PRLuNV6jTEbSDGMTEW174=
```

## 包名确认

接下来我们需要构造一个新的dex，嫁接到豌豆荚的apk前面；这里需要确认豌豆荚使用的包名：`com.wandoujia.phoenix2`

```
package: name='com.wandoujia.phoenix2' versionCode='16861' versionName='5.68.21'
sdkVersion:'14'
targetSdkVersion:'16'
```

另外值得一提的是，豌豆荚的权限还是比较`流氓`的会要求大量敏感权限，因此在使用该市场的时候注意权限的问题，否则很有可能裸奔了：

比如读/写短信，读/写通讯录等等，还有一些第三方权限

```
uses-permission:'android.permission.READ_SMS'
uses-permission:'android.permission.RECEIVE_SMS'
uses-permission:'android.permission.MANAGE_ACCOUNTS'
uses-permission:'android.permission.AUTHENTICATE_ACCOUNTS'
uses-permission:'android.permission.USE_CREDENTIALS'
uses-permission:'android.permission.READ_SETTINGS'
uses-permission:'android.permission.READ_EXTERNAL_STORAGE'
uses-permission:'android.permission.SEND_SMS'
uses-permission:'android.permission.WRITE_EXTERNAL_STORAGE'
uses-permission:'android.permission.MOUNT_UNMOUNT_FILESYSTEMS'
uses-permission:'android.permission.INTERNET'
uses-permission:'android.permission.ACCESS_NETWORK_STATE'
uses-permission:'android.permission.ACCESS_WIFI_STATE'
uses-permission:'android.permission.CHANGE_WIFI_STATE'
uses-permission:'android.permission.CHANGE_WIFI_MULTICAST_STATE'
uses-permission:'android.permission.SET_WALLPAPER'
uses-permission:'android.permission.SET_WALLPAPER_HINTS'
uses-permission:'android.permission.WRITE_SETTINGS'
uses-permission:'android.permission.CAMERA'
uses-permission:'android.permission.FLASHLIGHT'
uses-permission:'com.android.launcher.permission.INSTALL_SHORTCUT'
uses-permission:'com.android.launcher.permission.UNINSTALL_SHORTCUT'
uses-permission:'android.permission.READ_PHONE_STATE'
uses-permission:'android.permission.MODIFY_AUDIO_SETTINGS'
uses-permission:'android.permission.SYSTEM_ALERT_WINDOW'
uses-permission:'android.permission.ACCESS_SUPPERUSER'
uses-permission:'android.permission.GET_PACKAGE_SIZE'
uses-permission:'android.permission.KILL_BACKGROUND_PROCESSES'
uses-permission:'android.permission.CLEAR_APP_CACHE'
uses-permission:'android.permission.DISABLE_KEYGUARD'
uses-permission:'com.android.launcher.permission.READ_SETTINGS'
uses-permission:'com.android.launcher.permission.WRITE_SETTINGS'
uses-permission:'com.android.launcher3.permission.READ_SETTINGS'
uses-permission:'com.android.launcher3.permission.WRITE_SETTINGS'
uses-permission:'com.meizu.flyme.launcher.permission.READ_SETTINGS'
uses-permission:'com.meizu.flyme.launcher.permission.WRITE_SETTINGS'
uses-permission:'org.adw.launcher.permission.READ_SETTINGS'
uses-permission:'org.adw.launcher.permission.WRITE_SETTINGS'
uses-permission:'com.qihoo360.launcher.permission.READ_SETTINGS'
uses-permission:'com.qihoo360.launcher.permission.WRITE_SETTINGS'
uses-permission:'com.lge.launcher.permission.READ_SETTINGS'
uses-permission:'com.lge.launcher.permission.WRITE_SETTINGS'
uses-permission:'net.qihoo.launcher.permission.READ_SETTINGS'
uses-permission:'net.qihoo.launcher.permission.WRITE_SETTINGS'
uses-permission:'org.adwfreak.launcher.permission.READ_SETTINGS'
uses-permission:'org.adwfreak.launcher.permission.WRITE_SETTINGS'
uses-permission:'com.huawei.launcher3.permission.READ_SETTINGS'
uses-permission:'com.huawei.launcher3.permission.WRITE_SETTINGS'
uses-permission:'com.fede.launcher.permission.READ_SETTINGS'
uses-permission:'com.fede.launcher.permission.WRITE_SETTINGS'
uses-permission:'com.sec.android.app.twlauncher.settings.READ_SETTINGS'
uses-permission:'com.sec.android.app.twlauncher.settings.WRITE_SETTINGS'
uses-permission:'com.anddoes.launcher.permission.READ_SETTINGS'
uses-permission:'com.anddoes.launcher.permission.WRITE_SETTINGS'
uses-permission:'com.lenovo.launcher.permission.READ_SETTINGS'
uses-permission:'com.lenovo.launcher.permission.WRITE_SETTINGS'
uses-permission:'com.google.android.launcher.permission.READ_SETTINGS'
uses-permission:'com.google.android.launcher.permission.WRITE_SETTINGS'
uses-permission:'com.oppo.launcher.permission.WRITE_SETTINGS'
uses-permission:'com.oppo.launcher.permission.READ_SETTINGS'
uses-permission:'com.yulong.android.launcher3.permission.WRITE_SETTINGS'
uses-permission:'com.yulong.android.launcher3.permission.READ_SETTINGS'
uses-permission:'com.huawei.android.launcher.permission.READ_SETTINGS'
uses-permission:'com.huawei.android.launcher.permission.WRITE_SETTINGS'
uses-permission:'com.htc.launcher.permission.READ_SETTINGS'
uses-permission:'com.htc.launcher.permission.WRITE_SETTINGS'
uses-permission:'com.bbk.launcher2.permission.READ_SETTINGS'
uses-permission:'com.bbk.launcher2.permission.WRITE_SETTINGS'
uses-permission:'android.permission.WAKE_LOCK'
uses-permission:'android.permission.BROADCAST_PACKAGE_ADDED'
uses-permission:'android.permission.BROADCAST_PACKAGE_CHANGED'
uses-permission:'android.permission.BROADCAST_PACKAGE_INSTALL'
uses-permission:'android.permission.BROADCAST_PACKAGE_REPLACED'
uses-permission:'android.permission.RESTART_PACKAGES'
uses-permission:'android.permission.GET_TASKS'
uses-permission:'android.permission.RECEIVE_BOOT_COMPLETED'
uses-permission:'android.permission.CHANGE_NETWORK_STATE'
uses-permission:'android.permission.GET_ACCOUNTS'
uses-permission:'android.permission.VIBRATE'
uses-permission:'android.permission.BIND_ACCESSIBILITY_SERVICE'
uses-permission:'android.permission.READ_CONTACTS'
uses-permission:'android.permission.WRITE_CONTACTS'
uses-permission:'android.permission.CALL_PHONE'
uses-permission:'android.permission.WRITE_SMS'
uses-permission:'android.permission.WRITE_CALL_LOG'
uses-permission:'android.permission.READ_CALL_LOG'
uses-permission:'android.permission.AUTHENTICATE_ACCOUNTS'
uses-permission:'android.permission.WRITE_SYNC_SETTINGS'
uses-permission:'android.permission.MANAGE_ACCOUNTS'
uses-permission:'android.permission.ACCESS_FINE_LOCATION'
uses-permission:'android.permission.ACCESS_COARSE_LOCATION'
uses-permission:'com.wandoujia.phoenix2.permission.MIPUSH_RECEIVE'
uses-permission:'android.permission.PACKAGE_USAGE_STATS'
uses-permission:'android.permission.PERSISTENT_ACTIVITY'
uses-permission:'android.permission.ACCESS_MTK_MMHW'
```

## Hack

接下来开始编码工作，明确下我们的目标：

* 替换Application，并且在app进程启动时弹出一个toast；
* 替换启动页，显示一个特殊文案；

因此首先安确认下豌豆荚的自定义application：`com.pp.assistant.PPApplication`。

```
aven$ aapt dump xmltree Wandoujia_224660_web_inner_referral_binded.apk AndroidManifest.xml|less

    E: application (line=155)
      A: android:theme(0x01010000)=@0x7f0a0001
      A: android:label(0x01010001)=@0x7f0c038a
      A: android:icon(0x01010002)=@0x7f02009d
      A: android:name(0x01010003)="com.pp.assistant.PPApplication" (Raw: "com.pp.assistant.PPApplication")
      A: android:stateNotNeeded(0x01010016)=(type 0x12)0xffffffff
      A: android:windowSoftInputMode(0x0101022b)=(type 0x11)0x3
      A: android:allowBackup(0x01010280)=(type 0x12)0xffffffff
```

创建同名的PPApplication的，然后加上toast即可，接下来编译得到新的apk，并将其中的dex抽离出来备用。

## 插曲

在实际插入dex的时候，遇到了一些小插曲，比如插入完后，启动崩溃，所以如果是插入全新的dex的话，需要确认和原有dex的关系，如果完全摒弃原有逻辑，那么需要手动补全manifest中声明的`ContentProvider`和`BroadcastReceiver`，Activity根据需要替换，Service可选替换

另外合并apk和dex不是简单的字节叠加，需要修改最终apk的偏移量，确保zip的正确性。笔者使用的是一个Python脚本

```
https://github.com/V-E-O/PoC/tree/master/CVE-2017-13156
```

## 效果

搞定之后，可以直接安装apk，也可以覆盖升级安装，接下来启动app就可以看到完全不同的效果；

在这里我们出于实验性质，将豌豆荚市场的application和启动Activity做了整体替换，因此直接感受就是原有逻辑全部没有了，如果我们通过反编译后增量修改的方式来新增dex，纳闷可以实现和原app功能几乎一致的串改，这样可以恶意插入代码，同时不容易被用户发现。

![效果](/assets/img/device-2017-12-25-130147.gif)

## 修复

这个bug看起来挺严重的，不过实际上影响有限，如果用户通过正规市场下载程序基本没什么问题，同时Android官方已经做fix，相信后续很快就会在新版本中生效。
对于开发者来说比较被动，最好升级签名为V2，别的就没有屏蔽办法了，比较问题出在系统校验上面。
