---
layout: post
title: "vlc android 移植版编译"
description: "vlc android compiling"
category: 
tags: [media,VLC]
---
{% include JB/setup %}

##安装必备工具/解决环境问题

环境准备什么的如果没配置过需要一步步配置，主要是sdk/ndk，以及一些编译过程中需要用到的命令工具。
	* Requirements
		
	You MUST build on Linux (or OSX if you know what you are doing).
	The following packages MUST must be installed:
	* the GNU autotools: autoconf, libtool, automake and make (a.k.a. gmake)
	* ...and their dependencies: m4 and gawk, mawk or nawk,
	* the GNU C and C++ compilers a.k.a. gcc and g++,
	* some GNU build utilities: pkg-config and patch,
	* the following other build utilities: Apache Ant (or Ant), cmake, protobuf, ragel,
	* the Subversion and Git version control systems
	* unzip and either curl or wget for retreiving sources.
	* Very recent versions of some of those tools may be required. At the time of writing, notably gettext 0.19.3 or later is required.
	If any of the above is missing, expect the build to fail at some point.
	If targeting an Android-x86 device, yasm must be installed too.

特别要注意gettext的版本问题，系统如果自带了可能版本不一致，比如笔者的时0.18.3，这个时候可以进行升级，可利用port，如果原版本是用homebrew安装的则也可以通过brew升级，但是port安装的不能通过brew升级，这样的操作会导致本地安装一个新的gettext,并未覆盖原有的版本

	port upgrade outdated
	
##获取源码/编译
接着开始获取源代码，尝试编译；

获取源代码 

	git clone git://git.videolan.org/vlc-ports/android.git vlc-android


很遗憾第一次失败了：

	aven-mac-pro:vlc-android aven$ ./compile.sh 
	*** No ANDROID_ABI defined architecture: using ARMv7
	Downloading gradle
	  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
	                                 Dload  Upload   Total   Spent    Left  Speed
	  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
	Archive:  gradle-2.2.1-all.zip
	  End-of-central-directory signature not found.  Either this file is not
	  a zipfile, or it constitutes one disk of a multi-part archive.  In the
	  latter case the central directory and zipfile comment will be found on
	  the last disk(s) of this archive.
	note:  gradle-2.2.1-all.zip may be a plain executable, not an archive
	unzip:  cannot find zipfile directory in one of gradle-2.2.1-all.zip or
	        gradle-2.2.1-all.zip.zip, and cannot find gradle-2.2.1-all.zip.ZIP, period.

##错误处理
从异常信息来看应该是gradle-2.2.1-all.zip文件无效，导致文件校验失败。可以手工检测下gradle文件的内容：

	aven-mac-pro:vlc-android aven$ unzip -l gradle-2.2.1-all.zip 
	Archive:  gradle-2.2.1-all.zip
	  End-of-central-directory signature not found.  Either this file is not
	  a zipfile, or it constitutes one disk of a multi-part archive.  In the
	  latter case the central directory and zipfile comment will be found on
	  the last disk(s) of this archive.
	note:  gradle-2.2.1-all.zip may be a plain executable, not an archive
	unzip:  cannot find zipfile directory in one of gradle-2.2.1-all.zip or
	        gradle-2.2.1-all.zip.zip, and cannot find gradle-2.2.1-all.zip.ZIP, period.

果然失败，目测应该是网络下载文件失败，尝试翻墙后重新下载，或从其他项目拷贝一份，gradle官网罗列了所有可供下载的版本:[https://services.gradle.org/distributions](https://services.gradle.org/distributions)

下载完毕后解压为gradle/*，和AndroidStudio的格式保持一致，这个就是用来生成AndroidStudio配置使用的；

接下来脚本会自动尝试下载vlc源码，若有本地已有vlc源码可以拷贝一份vlc，因为下载比较慢

再次编译，遇到protobuf的缺失问题，port或brew安装之；

	protoc not found
	To-be-built packages: protoc
	rm -f -R protobuf   && tar xvjf protobuf-2.6.0.tar.bz2  
	mv protobuf-2.6.0 protobuf && touch protobuf
	mv: rename protobuf-2.6.0 to protobuf: No such file or directory
	make: *** [protobuf] Error 1
	buildsystem tools: make

安装过程最好保持翻墙状态，否则容易出现个别地址无法访问，例如amazon的云服务

	(7) Failed to connect to s3.amazonaws.com port 443: Operation timed out
 
 
 
解决完错误，继续编译，看到这段输出时基本也就大功告成：


	make: Entering directory `/Users/aven/work/video/vlc-android/libvlc'
	[armeabi-v7a] Gdbserver      : [arm-linux-androideabi-4.8] libs/armeabi-v7a/gdbserver
	[armeabi-v7a] Gdbsetup       : libs/armeabi-v7a/gdb.setup
	[armeabi-v7a] Install        : libanw.10.so => libs/armeabi-v7a/libanw.10.so
	[armeabi-v7a] Install        : libanw.13.so => libs/armeabi-v7a/libanw.13.so
	[armeabi-v7a] Install        : libanw.14.so => libs/armeabi-v7a/libanw.14.so
	[armeabi-v7a] Install        : libanw.18.so => libs/armeabi-v7a/libanw.18.so
	[armeabi-v7a] Install        : libanw.21.so => libs/armeabi-v7a/libanw.21.so
	[armeabi-v7a] Install        : libiomx.10.so => libs/armeabi-v7a/libiomx.10.so
	[armeabi-v7a] Install        : libiomx.13.so => libs/armeabi-v7a/libiomx.13.so
	[armeabi-v7a] Install        : libiomx.14.so => libs/armeabi-v7a/libiomx.14.so
	[armeabi-v7a] Compile thumb  : vlcjni <= libvlcjni.c
	[armeabi-v7a] SharedLibrary  : libvlcjni.so
	[armeabi-v7a] Install        : libvlcjni.so => libs/armeabi-v7a/libvlcjni.so
	make: Leaving directory `/Users/aven/work/video/vlc-android/libvlc'

	//省略部分

	:vlc-android:processVanillaARMv7DebugManifest UP-TO-DATE
	:vlc-android:processVanillaARMv7DebugResources UP-TO-DATE
	:vlc-android:generateVanillaARMv7DebugSources UP-TO-DATE
	:vlc-android:processVanillaARMv7DebugJavaRes UP-TO-DATE
	:vlc-android:compileVanillaARMv7DebugJava UP-TO-DATE
	:vlc-android:compileVanillaARMv7DebugNdk UP-TO-DATE
	:vlc-android:compileVanillaARMv7DebugSources UP-TO-DATE
	:vlc-android:preDexVanillaARMv7Debug
	:vlc-android:dexVanillaARMv7Debug
	:vlc-android:validateDebugSigning
	:vlc-android:packageVanillaARMv7Debug
	:vlc-android:zipalignVanillaARMv7Debug
	:vlc-android:assembleVanillaARMv7Debug
	
	BUILD SUCCESSFUL
	
	Total time: 23.73 secs

检查并安装demo：

	aven-mac-pro:vlc-android aven$ ls -al vlc-android/build/outputs/apk/
	total 53856
	drwxr-xr-x  4 aven  staff       136 Aug  1 12:34 .
	drwxr-xr-x  4 aven  staff       136 Aug  1 12:34 ..
	-rw-r--r--  1 aven  staff  13785101 Aug  1 12:34 VLC-Android-1.5.0-ARMv7.apk
	-rw-r--r--  1 aven  staff  13783961 Aug  1 12:34 vlc-android-vanilla-ARMv7-debug-unaligned.apk
	
	aven-mac-pro:vlc-android aven$ adb install vlc-android/build/outputs/apk/VLC-Android-1.5.0-ARMv7.apk 
	5836 KB/s (13785101 bytes in 2.306s)
        pkg: /data/local/tmp/VLC-Android-1.5.0-ARMv7.apk
	Success

##检验成果
运行demo，功能还是很强大的，基本上就是一个完善的播放器，还支持一些实用的功能，比如倍速播放；

![](http://7u2jir.com1.z0.glb.clouddn.com/device-2015-08-02-112218.png)
![](http://7u2jir.com1.z0.glb.clouddn.com/device-2015-08-02-112247.png)

##2015/11/02更新
vlc-android初次编译后，本地已经有了相关源代码，此时如果git pull更新代码，有可能再次编译时会报错，原因在于以来的仓库没有更新；

	Error: Your vlc checkout does not contain the latest tested commit:${TESTED_HASH}

比如上面的日志就是因为多个仓库的不一致引起；

具体脚本位于compile.sh

{% highlight bash %}
TESTED_HASH=ed96e80
if [ ! -d "vlc" ]; then
    echo "VLC source not found, cloning"
    git clone git://git.videolan.org/vlc.git vlc
    checkfail "vlc source: git clone failed"
else
    echo "VLC source found"
    cd vlc
    if ! git cat-file -e ${TESTED_HASH}; then
        cat << EOF
***
*** Error: Your vlc checkout does not contain the latest tested commit: ${TESTED_HASH}
***
EOF
        exit 1
    fi
    cd ..
fi

{%endhighlight%}

所以如果有更新代码需要确保各仓库更新是同步的；

###参考文档

* [https://wiki.videolan.org/AndroidCompile/](https://wiki.videolan.org/AndroidCompile/)