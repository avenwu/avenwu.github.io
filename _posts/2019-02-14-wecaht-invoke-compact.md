---
layout: post
title: "微信吊起app适配"
description: ""
header_image: "/assets/images/Screenshot_2019-02-14-19-18-07-412_com.android.sy.png"
keywords: ""
tags: []
---
{% include JB/setup %}

## 0x1 背景

最近微信的一些变化导致从微信内调起app开始水土不服，过去我们可以通过scheme直接吊起三方app，现在不行了。通过接口调整后，每次吊起app都会新建一个栈，导致菜单页面出现多个相同app记录，这是我们不希望看到的。

![效果图](/assets/images/Screenshot_2019-02-14-19-18-07-412_com.android.sy.png)


## 0x2 现状分析

启动一个Activity可以设置launchMode和FLAG，新版微信推测在吊起三方app是走的是类似于分享的处理，直接吊起了WXEntryActivity,并且给他添加了类似NEW_TASK, MULTI_TASK的flag。

判断一个App当前有多少个Task，可以通过dumpsys查看，清楚明了：

```shell
aven-mac-pro-2:~ aven$ adb shell dumpsys activity activities|grep TaskRecord|grep com.wuba|grep \*
    * TaskRecord{9fa5181 #5400 A=com.wuba U=0 StackId=1 sz=1}
```

也可以添加日志观察app的栈情况，比如可以统一添加activity启动的日志，输出当前taskid
```java
LOGGER.i("ActivityTrack", activity.getClass().getSimpleName() + " create, task id=" + activity.getTaskId());
```
多跑几遍吊起流程，会发现id一致在变化中
```
WXEntryActivity create, task id=5438
```

## 0x3 解决方案

为了规避这个问题，需要处理管理activity栈，确保落地页Activity所在的栈不会每次吊起都新建一个独立的。

### 0x3.1 销毁栈
可以在WXEntryActivity吊起落地页的时候，将当前栈清空并退出，以便后续Activity能够进入默认栈。
```java
finishAndRemoveTask()
```
这个办法有兼容性问题，因为该API是21字后引入的。

### 0x3.2 指定affinity

第二个办法，给WXEntryActivity配置一个独立的affinity，如此，其他所有页面都是默认值便能和WXEntryActivity进入不同栈。

```xml
<activity
    android:name="${applicationId}.wxapi.WXEntryActivity"
    android:configChanges="keyboardHidden|locale"
    android:exported="true"
    android:screenOrientation="portrait"
    android:taskAffinity="com.wuba.share"
    android:theme="@style/Theme.Translucent" />

```

## 0x4 采坑

### 0x4.1 正式包和Debug包

上面两张方案在测试的时候会遇到一个问题，从微信内吊起三方app，如果首次失败，那么后续一致会失败，原因是测试app的签名不一致。
要解决，只能清除微信数据，卸载重装微信。并且安装release的app

### 0x4.2 测试流程复杂，不可调试

这个问题其实和上一个本质相同，我们在得出上述解决方案之前，经历了大量测试验证，每次都需要发布签名是在太麻烦，笔者开发汇中即使改了一行代码需要重走流程到测试观察日志需要20分钟以上。

为了解决测试耗时和不能调试的问题，我们需要模拟微信吊起app。


## 0x5 模拟微信吊起app

通过dump当前的应用状态，我们可以知道微信吊起确实是唤起了WXEntryActivity，清切intent携带了一些数据。
我们需要确定传递了什么数据

```java
LOGGER.d("WeChat", getIntent().getExtras().toString());
```

有了这些数据，我们便可以模拟吊起Activity，来规避微信错误提示。

### 0x5.1 ADB调起WXEntryActivity

尝试通过adb启动Activity，并逐一添加参数

```
adb shell am start-activity -n com.wuba/com.wuba.wxapi.WXEntryActivity --es _wxappextendobject_extInfo xxxx
```

很快页面吊起来了，但是并没有直达落地页，而是停留在入口Activity上。
直觉反应肯定是参数没有通过校验，仔细核查所以参数，发现可以字段_mmessage_checksum

从命名上可以推测他是某种消息摘要，用于校验传递的数据，防止篡改。

### 0x5.2 参数分析

如果这个摘要有时间戳相关维度判断，那估计就没戏了，根据关键字，我们可以查一下微信sdk内部校验逻辑，关键代码如下：

```java
String var3 = var1.getStringExtra("_mmessage_content");
int var4 = var1.getIntExtra("_mmessage_sdkVersion", 0);
String var5;
if ((var5 = var1.getStringExtra("_mmessage_appPackage")) == null || var5.length() == 0) {
    Log.e("MicroMsg.SDK.WXApiImplV10", "invalid argument");
    return false;
}

byte[] var6 = var1.getByteArrayExtra("_mmessage_checksum");
byte[] var14 = b.a(var3, var4, var5);
if (!this.checkSumConsistent(var6, var14)) {
    Log.e("MicroMsg.SDK.WXApiImplV10", "checksum fail");
    return false;
}
```

```java
public final class b {
    public static final String e(byte[] var0) {
        char[] var1 = new char[]{'0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'a', 'b', 'c', 'd', 'e', 'f'};

        try {
            MessageDigest var2;
            (var2 = MessageDigest.getInstance("MD5")).update(var0);
            int var8;
            char[] var3 = new char[(var8 = (var0 = var2.digest()).length) * 2];
            int var4 = 0;

            for(int var5 = 0; var5 < var8; ++var5) {
                byte var6 = var0[var5];
                var3[var4++] = var1[var6 >>> 4 & 15];
                var3[var4++] = var1[var6 & 15];
            }

            return new String(var3);
        } catch (Exception var7) {
            return null;
        }
    }
}

private boolean checkSumConsistent(byte[] var1, byte[] var2) {
    if (var1 != null && var1.length != 0 && var2 != null && var2.length != 0) {
        if (var1.length != var2.length) {
            Log.e("MicroMsg.SDK.WXApiImplV10", "checkSumConsistent fail, length is different");
            return false;
        } else {
            for(int var3 = 0; var3 < var1.length; ++var3) {
                if (var1[var3] != var2[var3]) {
                    return false;
                }
            }

            return true;
        }
    } else {
        Log.e("MicroMsg.SDK.WXApiImplV10", "checkSumConsistent fail, invalid arguments");
        return false;
    }
}

```

分析上述代码，可知消息校验没有时间戳，只是一个MD5的HEX字符串判断。

同时，结合日志信息，发现几处问题:
* 参数类型错误
* byte[]数组问题

```
02-13 17:39:09.888 12938 12938 D MicroMsg.SDK.MMessage: send mm message, intent=Intent { act=com.tencent.mm.plugin.openapi.Intent.ACTION_HANDLE_APP_REGISTER (has extras) }, perm=com.tencent.mm.permission.MM_MESSAGE
02-13 17:39:09.888 12938 12938 W Bundle  : Key _mmessage_sdkVersion expected Integer but value was a java.lang.String.  The default value 0 was returned.
02-13 17:39:09.889 12938 12938 W Bundle  : Attempt to cast generated internal exception:
02-13 17:39:09.889 12938 12938 W Bundle  : java.lang.ClassCastException: java.lang.String cannot be cast to java.lang.Integer
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.os.BaseBundle.getInt(BaseBundle.java:1045)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.content.Intent.getIntExtra(Intent.java:7398)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at com.tencent.mm.opensdk.openapi.WXApiImplV10.handleIntent(Unknown Source:73)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at com.wuba.loginsdk.wxapi.WXCallbackEntryActivity.onCreate(WXCallbackEntryActivity.java:81)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at com.wuba.wxapi.WXEntryActivity.onCreate(WXEntryActivity.java:39)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.app.Activity.performCreate(Activity.java:7210)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.app.Activity.performCreate(Activity.java:7201)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1272)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2926)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:3081)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.app.servertransaction.LaunchActivityItem.execute(LaunchActivityItem.java:78)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.app.servertransaction.TransactionExecutor.executeCallbacks(TransactionExecutor.java:108)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:68)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1831)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.os.Handler.dispatchMessage(Handler.java:106)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.os.Looper.loop(Looper.java:201)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.app.ActivityThread.main(ActivityThread.java:6806)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at java.lang.reflect.Method.invoke(Native Method)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:547)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:873)
02-13 17:39:09.889 12938 12938 W Bundle  : Key _mmessage_checksum expected byte[] but value was a java.lang.String.  The default value <null> was returned.
02-13 17:39:09.889 12938 12964 I ContentCatcher: Interceptor : Catcher list invalid for com.wuba@com.wuba.wxapi.WXEntryActivity@246059734
02-13 17:39:09.889 12938 12964 I ContentCatcher: Interceptor : Get featureInfo from config pick_mode
02-13 17:39:09.889 12938 12938 W Bundle  : Attempt to cast generated internal exception:
02-13 17:39:09.889 12938 12938 W Bundle  : java.lang.ClassCastException: java.lang.String cannot be cast to byte[]
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.os.BaseBundle.getByteArray(BaseBundle.java:1357)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.os.Bundle.getByteArray(Bundle.java:1090)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.content.Intent.getByteArrayExtra(Intent.java:7607)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at com.tencent.mm.opensdk.openapi.WXApiImplV10.handleIntent(Unknown Source:105)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at com.wuba.loginsdk.wxapi.WXCallbackEntryActivity.onCreate(WXCallbackEntryActivity.java:81)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at com.wuba.wxapi.WXEntryActivity.onCreate(WXEntryActivity.java:39)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.app.Activity.performCreate(Activity.java:7210)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.app.Activity.performCreate(Activity.java:7201)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1272)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2926)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:3081)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.app.servertransaction.LaunchActivityItem.execute(LaunchActivityItem.java:78)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.app.servertransaction.TransactionExecutor.executeCallbacks(TransactionExecutor.java:108)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:68)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1831)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.os.Handler.dispatchMessage(Handler.java:106)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.os.Looper.loop(Looper.java:201)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at android.app.ActivityThread.main(ActivityThread.java:6806)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at java.lang.reflect.Method.invoke(Native Method)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:547)
02-13 17:39:09.889 12938 12938 W Bundle  : 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:873)
02-13 17:39:09.889 12938 12938 E MicroMsg.SDK.WXApiImplV10: checkSumConsistent fail, invalid arguments
```

由于adb不支持所以参数，比如序列化对象，字节数组等，因此通过adb启动并不可行

### 0x5.3 App调起WXEntryActivity

既然adb有限制，就换成一个测试app来吊起入口类。
```java
public void onClickOpen(View view) {
    Intent intent = new Intent();
    intent.setClassName("com.wuba", "com.wuba.wxapi.WXEntryActivity");
    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    intent.addFlags(Intent.FLAG_ACTIVITY_MULTIPLE_TASK);
    intent.putExtra("_wxappextendobject_extInfo", "wbmain://jump/core/common?params=%7B%22url%22%3A%22https%3A%2F%2Fdown.58.com%2Fh5%2F58person.html%22%2C%22title%22%3A%2258%E4%BA%BA%E7%89%A9%22%2C%22pagetype%22%3A%22common%22%7D");
    intent.putExtra("_wxapi_basereq_transaction", "f3ac1733e4b2154d97dc3996bff55287");
    intent.putExtra("_wxobject_sdkVer", 620953856);
    intent.putExtra("_mmessage_appPackage", "com.tencent.mm");
    intent.putExtra("_wxobject_message_ext", "wbmain://jump/core/common?params=%7B%22url%22%3A%22https%3A%2F%2Fdown.58.com%2Fh5%2F58person.html%22%2C%22title%22%3A%2258%E4%BA%BA%E7%89%A9%22%2C%22pagetype%22%3A%22common%22%7D");
    intent.putExtra("_wxapi_command_type", 4);
    intent.putExtra("_wxapi_basereq_openid", "");
    intent.putExtra("_mmessage_checksum", "1c84447a3580a34a094a9aa00820631a".getBytes());
    intent.putExtra("wx_token_key", "com.tencent.mm.openapi.token");
    intent.putExtra("_mmessage_sdkVersion", 620953856);
    intent.putExtra("_wxapi_showmessage_req_country", "CN");
    intent.putExtra("_wxobject_identifier_", "com.tencent.mm.sdk.openapi.WXAppExtendObject");
    intent.putExtra("_wxapi_showmessage_req_lang", "zh_CN");
    intent.putExtra("platformId", "wechat");
    startActivity(intent);
}
```

至此，我们可以绕开微信对app签名的限制，通过一个测试app，无限制吊起目标app，而数据则全部源自正是调用的样本。可以很方便的对app做调试。

