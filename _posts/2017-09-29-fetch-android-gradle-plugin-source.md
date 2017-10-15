---
layout: post
title: "为何获取Android Gradle Plugin源码这么麻烦？"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2017-09-30-01.jpg
keywords: ""
tags: []
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2017-09-30-01.jpg)

## 背景

已经忘了多少次想获取Android Gradle 的源码，但是多少次又搁置了。

## repo sync

在使用Android Studio的时候，需要配置一个android的classpath这个大家应该不陌生，正如自定义的plugin配置类似。

```groovy
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.3.3'

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}
```

根据[tools](http://tools.android.com/build/gradleplugin)的说名，获取其源码是很简单的。

![gradle plugin](http://7u2jir.com1.z0.glb.clouddn.com/img/QQ20170929-175352@2x.png)

根据提示可以来到这个页面。

![checkout gradle source](http://7u2jir.com1.z0.glb.clouddn.com/img/QQ20170929-175443@2x.png)

这里看到了一段很熟悉的代码，早些年跟风获取Android的源代码就是用的这一套repo脚本。但是当时获取这些代码真的费了很长时间，如今时过境迁，Android都已经发展到8.0了，之前获取整套代码好像占用了20多G磁盘，如今估计要更多了。

如果网速够快，时间够充分，那么一直挂着总能下载完的（翻墙如果不方便可以找一个国内镜像）。

笔者就是在这遇到的坑，无论下多久，总是没完没了，我只不过想获取一下gradle plugin源码，为什么下载这么久都不行？

```
aven-mac-pro-2:gradle_2.3.0 aven$ repo sync
remote: Counting objects: 18, done.        
remote: Compressing objects: 100% (14/14), done.        
remote: Total 18 (delta 6), reused 0 (delta 0)        
From https://aosp.tuna.tsinghua.edu.cn/platform/manifest
   686970d..e3a2598  build-tools -> origin/build-tools
   b0584e9..a7aab94  emu-2.5-release -> origin/emu-2.5-release
   147e5f9..8985c11  master     -> origin/master
   fbcddeb..ab0f9cd  master-plus-llvm -> origin/master-plus-llvm
   1aa85e7..7394b30  oreo-cts-release -> origin/oreo-cts-release
Fetching project platform/tools/studio/google/cloud/testing
Fetching project platform/tools/idea
Fetching project platform/prebuilts/fullsdk/build-tools/21-darwin
Fetching project platform/prebuilts/ninja/darwin-x86
Fetching projects:   1% (1/66)  Fetching project platform/tools/analytics-library
Fetching projects:   3% (2/66)  Fetching project platform/external/smali
Fetching projects:   4% (3/66)  Fetching project platform/tools/external/go/src/golang.org/x/tools
Fetching projects:   6% (4/66)  Fetching project platform/prebuilts/clang/darwin-x86/host/3.5
Fetching projects:   7% (5/66)  Fetching project platform/external/boringssl
Fetching projects:   9% (6/66)  Fetching project platform/external/AntennaPod/afollestad
Fetching projects:  10% (7/66)  Fetching project platform/tools/gpu
Fetching projects:  12% (8/66)  Fetching project platform/prebuilts/gcc/darwin-x86/x86/x86_64-linux-android-4.9
Fetching projects:  13% (9/66)  Fetching project platform/external/AntennaPod/AntennaPod
remote: Counting objects: 2900123, done.        
Fetching projects:  15% (10/66)  Fetching project platform/tools/buildSrc
remote: Counting objects: 114, done.        
remote: Compressing objects: 100% (101/101), done.        
remote: Total 114 (delta 95), reused 27 (delta 11)        
Receiving objects: 100% (114/114), 84.34 KiB | 0 bytes/s, done.
Fetching projects:  16% (11/66)  Fetching project platform/tools/base
Resolving deltas: 100% (95/95), completed with 81 local objects.s   
From https://aosp.tuna.tsinghua.edu.cn/platform/external/boringssl
   db4251a..a24758f  master     -> aosp/master
Fetching projects:  18% (12/66)  Fetching project platform/prebuilts/gcc/linux-x86/host/x86_64-w64-mingw32-4.8
Fetching projects:  19% (13/66)  Fetching project platform/tools/studio/google/cloud/tools
remote: Counting objects: 157, done.        
remote: Compressing objects: 100% (52/52), done.        
Fetching projects:  21% (14/66)  Fetching project platform/prebuilts/studio/layoutlib
Fetching projects:  22% (15/66)  Fetching project platform/prebuilts/eclipse-build-deps
Fetching projects:  24% (16/66)  Fetching project platform/external/android-studio-gradle-test
remote: Counting objects: 549, done.        
Fetching projects:  25% (17/66)  Fetching project platform/tools/studio/google/services
Fetching projects:  27% (18/66)  Fetching project platform/tools/external/go/src/golang.org/x/net
Fetching projects:  28% (19/66)  Fetching project platform/tools/studio/google/appindexing
Fetching projects:  30% (20/66)  Fetching project platform/tools/swt
Fetching projects:  31% (21/66)  Fetching project platform/prebuilts/sdk
remote: Counting objects: 46522, done.        85.00 KiB/s   iB/s   
remote: Total 157 (delta 105), reused 139 (delta 97)             s   
Receiving objects: 100% (157/157), 26.82 MiB | 181.00 KiB/s, done.
Resolving deltas: 100% (105/105), done.
From https://aosp.tuna.tsinghua.edu.cn/platform/prebuilts/gcc/darwin-x86/x86/x86_64-linux-android-4.9
   9b83502..cb32df0  master     -> aosp/master
Fetching projects:  33% (22/66)  Fetching project repo
Fetching projects:  34% (23/66)  Fetching project platform/prebuilts/tools
remote: Counting objects: 42446, done.        iB | 154.00 KiB/s      
^Cerror: Cannot fetch platform/prebuilts/tools2 MiB | 277.00 KiB/s   
aborted by user
```
## 简化下载

实际上最诡异的就是每次都会发下执行同步命令后，出现一堆的android 历史版本的分支，tag，等等，还有其他一些完全不相关的仓库也在被下载。所以这里面肯定是有问题。

通过分析日志，我们可以知道，android用repo来同步代码的原因就是仓库太多，各种模块各种库，这些库从执行情况应该都是配置在了一个manifest钟。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>

  <remote  name="aosp"
           fetch=".."
           review="https://android-review.googlesource.com/" />
  <default revision="refs/tags/gradle_2.3.0"
           remote="aosp"
           sync-j="4" />

  <project path="tools/buildSrc" name="platform/tools/buildSrc">
    <copyfile src="base/build.gradle" dest="tools/build.gradle"/>
    <copyfile src="base/settings.gradle" dest="tools/settings.gradle"/>
    <copyfile src="base/gradlew" dest="tools/gradlew"/>
    <copyfile src="base/gradlew.bat" dest="tools/gradlew.bat"/>
    <copyfile src="base/gradle.properties" dest="tools/gradle.properties"/>
  </project>

  <project path="external/ant-glob" name="platform/external/ant-glob"/>
  <project path="external/boringssl" name="platform/external/boringssl"/>
  <project path="external/doclava" name="platform/external/doclava"/>
  <project path="external/gmock" name="platform/external/gmock" groups="gpu"/>
  <project path="external/gtest" name="platform/external/gtest"/>
  <project path="external/googletest" name="platform/external/googletest"/>
  <project path="external/iosched" name="platform/external/iosched"/>
  <project path="external/AntennaPod/AntennaPod" name="platform/external/AntennaPod/AntennaPod"/>
  <project path="external/AntennaPod/AudioPlayer" name="platform/external/AntennaPod/AudioPlayer"/>
  <project path="external/AntennaPod/afollestad" name="platform/external/AntennaPod/afollestad"/>
  <project path="external/gradle-perf-android-medium" name="platform/external/gradle-perf-android-medium"/>
  <project path="external/android-studio-gradle-test" name="platform/external/android-studio-gradle-test"/>
  <project path="external/protobuf" name="platform/external/protobuf"/>
  <project path="external/nanopb-c" name="platform/external/nanopb-c"/>
  <project path="external/grpc-grpc" name="platform/external/grpc-grpc"/>
  <project path="external/smali" name="platform/external/smali"/>
  <project path="external/zlib" name="platform/external/zlib"/>
  <project path="tools/analytics-library" name="platform/tools/analytics-library"/>
  <project path="tools/external/go/src/golang.org/x/net" name="platform/tools/external/go/src/golang.org/x/net"/>
  <project path="tools/external/go/src/golang.org/x/tools" name="platform/tools/external/go/src/golang.org/x/tools"/>
  <project path="frameworks/native" name="platform/frameworks/native"/>
  <project path="frameworks/base" name="platform/frameworks/base"  groups="notdefault,layoutlib"/>
  <project path="prebuilts/clang/darwin-x86/host/3.5" name="platform/prebuilts/clang/darwin-x86/host/3.5" groups="gpu,notdefault,platform-darwin"/>
  <project path="prebuilts/clang/darwin-x86/sdk/3.5" name="platform/prebuilts/clang/darwin-x86/sdk/3.5"/>
  <project path="prebuilts/clang/linux-x86/host/3.6" name="platform/prebuilts/clang/linux-x86/host/3.6" groups="lldb,notdefault,platform-linux"/>
  <project path="prebuilts/clang/host/linux-x86" name="platform/prebuilts/clang/host/linux-x86" groups="notdefault,platform-linux"/>
  <project path="prebuilts/clang/host/darwin-x86" name="platform/prebuilts/clang/host/darwin-x86" groups="notdefault,platform-darwin"/>
  <project path="prebuilts/cmake/darwin-x86" name="platform/prebuilts/cmake/darwin-x86" groups="lldb,notdefault,platform-darwin"/>
  <project path="prebuilts/cmake/linux-x86" name="platform/prebuilts/cmake/linux-x86" groups="lldb,notdefault,platform-linux"/>
  <project path="prebuilts/cmake/windows-x86" name="platform/prebuilts/cmake/windows-x86" groups="lldb,notdefault,platform-cygwin_nt-6.1,platform-cygwin_nt-6.2,platform-cygwin_nt-6.3,platform-cygwin_nt-10.0"/>
  <project path="prebuilts/eclipse-build-deps" name="platform/prebuilts/eclipse-build-deps"/>
  <project path="prebuilts/eclipse" name="platform/prebuilts/eclipse"/>
  <project path="prebuilts/maven_repo/android" name="platform/prebuilts/maven_repo/android"/>
  <project path="prebuilts/fullsdk/darwin/build-tools/21-darwin" name="platform/prebuilts/fullsdk/build-tools/21-darwin" groups="gpu,notdefault,platform-darwin"/>
  <project path="prebuilts/fullsdk/darwin/platforms/android-21" name="platform/prebuilts/fullsdk/platforms/android-21" groups="gpu,notdefault,platform-darwin"/>
  <project path="prebuilts/fullsdk/linux/build-tools/21-linux" name="platform/prebuilts/fullsdk/build-tools/21-linux" groups="gpu,notdefault,platform-linux"/>
  <project path="prebuilts/fullsdk/linux/platforms/android-21" name="platform/prebuilts/fullsdk/platforms/android-21" groups="gpu,notdefault,platform-linux"/>
  <project path="prebuilts/sdk" name="platform/prebuilts/sdk"/>
  <project path="prebuilts/gcc/darwin-x86/aarch64/aarch64-linux-android-4.9" name="platform/prebuilts/gcc/darwin-x86/aarch64/aarch64-linux-android-4.9" groups="gpu,notdefault,platform-darwin"/>
  <project path="prebuilts/gcc/darwin-x86/arm/arm-linux-androideabi-4.9" name="platform/prebuilts/gcc/darwin-x86/arm/arm-linux-androideabi-4.9" groups="gpu,notdefault,platform-darwin"/>
  <project path="prebuilts/gcc/darwin-x86/x86/x86_64-linux-android-4.9" name="platform/prebuilts/gcc/darwin-x86/x86/x86_64-linux-android-4.9" groups="gpu,notdefault,platform-darwin"/>
  <project path="prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9" name="platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9" groups="gpu,notdefault,platform-linux"/>
  <project path="prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.9" name="platform/prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.9" groups="gpu,notdefault,platform-linux"/>
  <project path="prebuilts/gcc/linux-x86/x86/x86_64-linux-android-4.9" name="platform/prebuilts/gcc/linux-x86/x86/x86_64-linux-android-4.9" groups="gpu,notdefault,platform-linux"/>
  <project path="prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.11-4.6" name="platform/prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.11-4.6" groups="notdefault,platform-linux"/>
  <project path="prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.11-4.8" name="platform/prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.11-4.8" groups="notdefault,platform-linux"/>
  <project path="prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.15-4.8" name="platform/prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.15-4.8" groups="lldb,notdefault,platform-linux"/>
  <project path="prebuilts/gcc/linux-x86/host/x86_64-w64-mingw32-4.8" name="platform/prebuilts/gcc/linux-x86/host/x86_64-w64-mingw32-4.8"/>
  <project path="prebuilts/go/linux-x86" name="platform/prebuilts/go/linux-x86" groups="gpu,notdefault,platform-linux"/>
  <project path="prebuilts/go/darwin-x86" name="platform/prebuilts/go/darwin-x86" groups="gpu,notdefault,platform-darwin"/>
  <project path="prebuilts/go/windows-x86" name="platform/prebuilts/go/windows-x86" groups="gpu,notdefault,platform-windows"/>
  <project path="prebuilts/google-breakpad/darwin-x86" name="platform/prebuilts/google-breakpad/darwin-x86" groups="lldb,notdefault,darwin"/>
  <project path="prebuilts/google-breakpad/linux-x86" name="platform/prebuilts/google-breakpad/linux-x86" groups="lldb,notdefault,linux"/>
  <project path="prebuilts/google-breakpad/windows-x86" name="platform/prebuilts/google-breakpad/windows-x86" groups="lldb,notdefault,windows"/>
  <project path="prebuilts/libedit/darwin-x86" name="platform/prebuilts/libedit/darwin-x86" groups="lldb,notdefault,darwin"/>
  <project path="prebuilts/libedit/linux-x86" name="platform/prebuilts/libedit/linux-x86" groups="lldb,notdefault,linux"/>
  <project path="prebuilts/libglog/darwin-x86" name="platform/prebuilts/libglog/darwin-x86" groups="lldb,notdefault,darwin"/>
  <project path="prebuilts/libglog/linux-x86" name="platform/prebuilts/libglog/linux-x86" groups="lldb,notdefault,linux"/>
  <project path="prebuilts/libglog/windows-x86" name="platform/prebuilts/libglog/windows-x86" groups="lldb,notdefault,windows"/>
  <project path="prebuilts/libprotobuf/darwin-x86" name="platform/prebuilts/libprotobuf/darwin-x86" groups="lldb,notdefault,darwin"/>
  <project path="prebuilts/libprotobuf/linux-x86" name="platform/prebuilts/libprotobuf/linux-x86" groups="lldb,notdefault,linux"/>
  <project path="prebuilts/libprotobuf/windows-x86" name="platform/prebuilts/libprotobuf/windows-x86" groups="lldb,notdefault,windows"/>
  <project path="prebuilts/ndk" name="platform/prebuilts/ndk" groups="gpu,notdefault,platform-linux,platform-darwin"/>
  <project path="prebuilts/ninja/darwin-x86" name="platform/prebuilts/ninja/darwin-x86" groups="lldb,notdefault,platform-darwin"/>
  <project path="prebuilts/ninja/linux-x86" name="platform/prebuilts/ninja/linux-x86" groups="lldb,notdefault,platform-linux"/>
  <project path="prebuilts/ninja/windows-x86" name="platform/prebuilts/ninja/windows-x86" groups="lldb,notdefault,platform-cygwin_nt-6.1,platform-cygwin_nt-6.2,platform-cygwin_nt-6.3,platform-cygwin_nt-10.0"/>
  <project path="prebuilts/python/linux-x86" name="platform/prebuilts/python/linux-x86" groups="lldb,notdefault,linux"/>
  <project path="prebuilts/python/windows-x86" name="platform/prebuilts/python/windows-x86" groups="lldb,notdefault,windows"/>
  <project path="prebuilts/studio/jdk" name="platform/prebuilts/studio/jdk"/>
  <project path="prebuilts/studio/layoutlib" name="platform/prebuilts/studio/layoutlib"/>
  <project path="prebuilts/swig/darwin-x86" name="platform/prebuilts/swig/darwin-x86" groups="lldb,notdefault,darwin"/>
  <project path="prebuilts/swig/linux-x86" name="platform/prebuilts/swig/linux-x86" groups="lldb,notdefault,linux"/>
  <project path="prebuilts/swig/windows-x86" name="platform/prebuilts/swig/windows-x86" groups="lldb,notdefault,windows"/>
  <project path="prebuilts/tools" name="platform/prebuilts/tools"/>
  <project path="sdk" name="platform/sdk"/>
  <project path="tools/adt/idea" name="platform/tools/adt/idea"/>
  <project path="tools/apksig" name="platform/tools/apksig"/>
  <project path="tools/dx/dalvik" name="platform/dalvik" groups="notdefault,platform-linux,platform-darwin"/>
  <project path="tools/dx/libcore" name="platform/libcore" groups="notdefault,platform-linux,platform-darwin"/>
  <project path="tools/base" name="platform/tools/base">
    <copyfile src="bazel/tools.idea.BUILD" dest="tools/BUILD"/>
    <copyfile src="bazel/toplevel.WORKSPACE" dest="WORKSPACE"/>
    <copyfile src="bazel/sdk/prebuilts.studio.sdk.BUILD" dest="prebuilts/studio/sdk/BUILD"/>
    <copyfile src="bazel/sdk/prebuilts.studio.sdk.README.md" dest="prebuilts/studio/sdk/README.md"/>
   </project>
  <project path="tools/data-binding" name="platform/frameworks/data-binding"/>
  <project path="tools/external/fat32lib" name="platform/tools/external/fat32lib"/>
  <project path="tools/external/gradle" name="platform/tools/external/gradle"/>
  <project path="tools/gpu/src/android.googlesource.com/platform/tools/gpu" name="platform/tools/gpu" groups="gpu"/>
  <project path="tools/gradle" name="platform/tools/gradle"/>
  <project path="tools/idea" name="platform/tools/idea"/>
  <project path="tools/repohooks" name="platform/tools/repohooks"/>
  <project path="tools/sherpa" name="platform/frameworks/opt/sherpa"/>
  <project path="tools/studio/google/cloud/tools" name="platform/tools/studio/google/cloud/tools"/>
  <project path="tools/studio/google/cloud/testing" name="platform/tools/studio/google/cloud/testing"/>
  <project path="tools/studio/google/login" name="platform/tools/studio/google/login"/>
  <project path="tools/studio/google/play" name="platform/tools/studio/google/play"/>
  <project path="tools/studio/google/samples" name="platform/tools/studio/google/samples"/>
  <project path="tools/studio/google/services" name="platform/tools/studio/google/services"/>
  <project path="tools/studio/google/appindexing" name="platform/tools/studio/google/appindexing"/>
  <project path="tools/swing-testing" name="platform/tools/swing-testing"/>
  <project path="tools/swt" name="platform/tools/swt"/>

</manifest>
```

所以虽然我们在根据官方说明制定了下载gradle分支，但是还是有很多冗余代码被下载了。由于我们并不是要编译源代码，也不会用到那么多的库，因此可以尝试精简仓库。

根据说明，我们大概知道gradle的代码在tool/base下面，因此我们只保留以下几个仓库代码，其他的全部删除：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>

  <remote  name="aosp"
           fetch=".."
           review="https://android-review.googlesource.com/" />
  <default revision="refs/tags/gradle_2.3.0"
           remote="aosp"
           sync-j="4" />
  <project path="tools/buildSrc" name="platform/tools/buildSrc">
    <copyfile src="base/build.gradle" dest="tools/build.gradle"/>
	<copyfile src="base/settings.gradle" dest="tools/settings.gradle"/>
    <copyfile src="base/gradlew" dest="tools/gradlew"/>
    <copyfile src="base/gradlew.bat" dest="tools/gradlew.bat"/>
	<copyfile src="base/gradle.properties" dest="tools/gradle.properties"/>
  </project>
  <project path="tools/adt/idea" name="platform/tools/adt/idea"/>
  <project path="tools/base" name="platform/tools/base">
    <copyfile src="bazel/tools.idea.BUILD" dest="tools/BUILD"/>
    <copyfile src="bazel/toplevel.WORKSPACE" dest="WORKSPACE"/>
    <copyfile src="bazel/sdk/prebuilts.studio.sdk.BUILD" dest="prebuilts/studio/sdk/BUILD"/>
    <copyfile src="bazel/sdk/prebuilts.studio.sdk.README.md" dest="prebuilts/studio/sdk/README.md"/>
  </project>

</manifest>
```

经过精简，再次repo sync，不出5分钟，代码下载完毕。

```
drwxr-xr-x  7 aven  staff   374 Sep 29 17:41 .repo
-r--r--r--  1 aven  staff  1377 Sep 29 17:31 WORKSPACE
drwxr-xr-x  3 aven  staff   102 Sep 29 17:31 prebuilts
drwxr-xr-x  5 aven  staff   374 Sep 29 17:41 tools
```

接下啦就可以愉快的开始源码分析之路了。

## 小结

在天朝，搞点东西还是有些费劲的，网速，和翻墙往往使得意见简单的事情变得更加复杂。

