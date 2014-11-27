---
layout: post
title: "Android Studio NDK配置"
tagline: ""
tags : [android,ndk]
---
{% include JB/setup %}

自行下载SDK与NDK，[NDK下载](https://developer.android.com/tools/sdk/ndk/index.html)
前两年如果要配置NDK，混合开发c/c++代码的话还需要自行辨析make文件，如Application.mk, Android.mk,实在不是很方便。
Android Studio目前已经支持c/c++代码编译，切自动生成所需make文件，同时还可以在build.gradle中配置原本卸载make里的参数。所以针对不太复杂的jni实现，用Android Studio会很方便。

在写第一个demo时可能还是会遇到一些问题，这里做一个分享，希望能帮到遇到类似问题的新手童鞋：


- "ndk配置"

> 在build.gradle内添加，具体在defaultConfig中：
>       ndk {
            moduleName "hello"
            stl "stlport_static"
        }

- "no rule to make target" 

> -这个问题据说是bug，是由于c/c++只有一个文件导致，所以添加一个空的c文件即可：touch empty.c

- "fatal error: string.h: No such file or directory"

> -这个原因可能比较多，但本质是路径不对，如果没用引用第三方的头文件，理论上不需要再做配置，这些头文件都在ndk的子目录中，所以在>local.properties中配置ndk.dir,类似sdk.dir. 另外有一个要注意的是版本问题，即build tool的版本，ndk需和sdk保持一致，如贵哦sdk用21，>ndk只有20，那么无论如何也成功不了，所以确保版本一致。


完整操作可以参考下面列出的参考网址。

###参考
1. https://software.intel.com/en-us/articles/building-native-android-apps-using-intelr-c-compiler-in-android-studio
2. http://stackoverflow.com/questions/9130429/android-ndk-build-iostream-no-such-file-or-directory
3. https://code.google.com/p/android/issues/detail?id=66937
