---
layout: post
title: "小试ijkplayer编译"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-16.jpg
description: ""
category: "ijkplayer" 
tags: [video]
---
{% include JB/setup %}

![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-16.jpg)

谈到视频播放大家都知道ffmpeg,基于其的衍生版本也很多，比如本文的ijkplayer.

## 试试ijkplayer编译

去到B站得github主页，找到ijkplayer项目，clone源码

	git clone git@github.com:Bilibili/ijkplayer.git

根据介绍文档一步步开始

## ./init-android.sh

执行初始化的shell脚本，脚本会自动下载ffmpeg的主干代码

{% highlight bash lineanchors %}

IJK_FFMPEG_UPSTREAM=git://git.videolan.org/ffmpeg.git
IJK_FFMPEG_FORK=https://github.com/Bilibili/FFmpeg.git
IJK_FFMPEG_COMMIT=ijk-r0.2.2-dev
IJK_FFMPEG_LOCAL_REPO=extra/ffmpeg

set -e
TOOLS=tools

echo "== pull ffmpeg base =="
sh $TOOLS/pull-repo-base.sh $IJK_FFMPEG_UPSTREAM $IJK_FFMPEG_LOCAL_REPO

function pull_fork()
{
    echo "== pull ffmpeg fork $1 =="
	sh $TOOLS/pull-repo-ref.sh $IJK_FFMPEG_FORK android/ffmpeg-$1 ${IJK_FFMPEG_LOCAL_REPO}
	cd android/ffmpeg-$1
	git checkout ${IJK_FFMPEG_COMMIT}
	cd -
}

pull_fork "armv7a"
pull_fork "armv5"
pull_fork "x86"
pull_fork "arm64-v8a"

./init-config.sh
./init-android-libyuv.sh

{% endhighlight %}

简单分析一息脚本做的事情，
首先定义了4个先关的变量，用于后文下载源码和目录的文件

	1. IJK_FFMPEG_UPSTREAM=git://git.videolan.org/ffmpeg.git
	ffmpeg的官方repo地址
	2. IJK_FFMPEG_FORK=https://github.com/Bilibili/FFmpeg.git
	B站托管与github的ffmpeg分支
	3. IJK_FFMPEG_COMMIT=ijk-r0.2.2-dev
	可以用来编译不同平台版本的代码分支

找到tools目录的pull-repo-ref.sh脚本，这个脚本用于获取B站托管的github上的FFmpeg分支,根据不同的目标平台放置不同工作目录,后面单独分析；

获取完代码后，检出之前定义的特定分支，也就是ijk-r0.2.2-dev
最后再调用另外两个脚本init-config.sh和init-android-libyuv.sh

## tools/pull-repo-ref.sh

这里脚本接受了三个参数

	REMOTE_REPO=https://github.com/Bilibili/FFmpeg.git B站自己托管站github的FFmpeg
	LOCAL_WORKSPACE=android/ffmpeg-xxx 本地目标仓库存放位置，xxx为平台版本
	REF_REPO=extra/ffmpeg 远程ffmpe官方仓库clone到本地的位置

接下来通过 git clone --reference来获取B站得FFmpeg,为什么这里的clone加了--reference参数？实际上根据用法介绍,--reference是为了减少从网络获取的文件，尽可能从本地的仓库中获取；

所以B站FFmpeg应该是基于官方的ffmpeg，这样的话，由于一开始的时候已经获取了官方ffmpeg代码，在获取B站的分支仓库时就可以将之前下好的官方仓库的本地库作为参考仓库;

{% highlight bash lineanchors %}

REMOTE_REPO=$1
LOCAL_WORKSPACE=$2
REF_REPO=$3

if [ -z $1 -o -z $2 -o -z $3 ]; then
    echo "invalid call pull-repo.sh '$1' '$2' '$3'"
elif [ ! -d $LOCAL_WORKSPACE ]; then
	git clone --reference $REF_REPO $REMOTE_REPO $LOCAL_WORKSPACE
	cd $LOCAL_WORKSPACE
	git repack -a
else
	cd $LOCAL_WORKSPACE
	git pull --rebase
	cd -
fi
{% endhighlight %}

## ./init-config.sh
这个脚本用来安全检查，如果config/module.sh不存在，那么默认将module-lite.sh拷贝作为module.sh，至于module.sh，在前面已经提到，主要是对标准ffmpeg的剪裁控制

{% highlight bash lineanchors %}
if [ ! -f 'config/module.sh' ]; then
    cp config/module-lite.sh config/module.sh
{% endhighlight %}

## ./init-android-libyuv.sh
这个脚本和前面提到的类似，只不过是用来下载依赖包的

{% highlight bash lineanchors %}
IJK_LIBYUV_UPSTREAM=https://github.com/Bilibili/libyuv.git
IJK_LIBYUV_FORK=https://github.com/Bilibili/libyuv.git
IJK_LIBYUV_COMMIT=ijk-r0.2.1-dev
IJK_LIBYUV_LOCAL_REPO=extra/libyuv

set -e
TOOLS=tools

echo "== pull libyuv base =="
sh $TOOLS/pull-repo-base.sh $IJK_LIBYUV_UPSTREAM $IJK_LIBYUV_LOCAL_REPO

echo "== pull libyuv fork =="
sh $TOOLS/pull-repo-ref.sh $IJK_LIBYUV_FORK ijkmedia/ijkyuv ${IJK_LIBYUV_LOCAL_REPO}
cd ijkmedia/ijkyuv
git checkout ${IJK_LIBYUV_COMMIT}
cd -
{% endhighlight %}

## 开始编译
初始化完毕后就可以依次执行下面的脚本开始编译了

	cd android
	./compile-ffmpeg.sh clean
	./compile-ffmpeg.sh
	./compile-ijk.sh

这里可能会遇到一些问题，根据自身环境不同不一定一样;笔者在编译时，第一次直接在master上编，结果提示有一个.h找不到，根据文档指南切回k0.2.3后重新来就OK了,虽然上面也说了可以在maste直接编，但实际上是不行的；


最后编译成功的话会看到几个so的输出


注意默认情况下这里之变异了armv7的几个so库，其他平台的需要跟上参数编译

	compile-ijk.sh armv5|armv7a|x86|arm64-v8a
	或者
	compile-ijk.sh all

但是，事情不是这么简单，但加上all之后,armv7仍然成功，但是x86编译就跪了
同时armv5也跪了，原因都是找不libijkffmpeg.so
冲armv7得输出日志可以看到,这个文件是build下从各自版本的编译文件拷贝而来;
定位到build文件夹，里面只有armv7的相关文件，所以根本拷贝就不会成功，自然就挂了;

{% highlight bash lineanchors %}

aven-mac-pro:android aven$ ./compile-ijk.sh all
Android NDK: ERROR:/Users/aven/work/video/ijkplayer/android/ijkmediaplayer-armv5/jni/ffmpeg/Android.mk:ijkffmpeg: LOCAL_SRC_FILES points to a missing file    
Android NDK: Check that /libijkffmpeg.so exists  or that its path is correct   
/Users/aven/Android/android-ndk-r10c/build/core/prebuilt-library.mk:45: *** Android NDK: Aborting    .  Stop.
/Users/aven/work/video/ijkplayer/android
[armeabi-v7a] Prebuilt       : libijkffmpeg.so <= /Users/aven/work/video/ijkplayer/android/build/ffmpeg-armv7a/output/
[armeabi-v7a] Install        : libijkffmpeg.so => libs/armeabi-v7a/libijkffmpeg.so
[armeabi-v7a] Compile thumb  : ijkplayer <= ff_cmdutils.c
[armeabi-v7a] Compile thumb  : ijkplayer <= ff_ffplay.c
[armeabi-v7a] Compile thumb  : ijkplayer <= ijkmeta.c
[armeabi-v7a] Compile thumb  : ijkplayer <= ijkplayer.c
[armeabi-v7a] Compile thumb  : ijkplayer <= ffpipeline_ffplay.c
[armeabi-v7a] Compile thumb  : ijkplayer <= ffpipenode_ffplay_vdec.c
[armeabi-v7a] Compile thumb  : ijkplayer <= ffpipenode_ffplay_vout.c
[armeabi-v7a] Compile thumb  : ijkplayer <= ffmpeg_api_jni.c
[armeabi-v7a] Compile thumb  : ijkplayer <= ijkplayer_android.c
[armeabi-v7a] Compile thumb  : ijkplayer <= ijkplayer_jni.c
[armeabi-v7a] Compile thumb  : ijkplayer <= ffpipeline_android.c
[armeabi-v7a] Compile thumb  : ijkplayer <= ffpipenode_android_mediacodec_vdec.c
[armeabi-v7a] Compile thumb  : ijkplayer <= ffpipenode_android_mediacodec_vout.c
[armeabi-v7a] Compile thumb  : ijksdl <= ijksdl_vout_overlay_ffmpeg.c
[armeabi-v7a] Compile thumb  : ijksdl <= image_convert.c
[armeabi-v7a] Compile thumb  : ijksdl <= android_nativewindow.c
[armeabi-v7a] Compile thumb  : ijksdl <= ijksdl_vout_android_nativewindow.c
[armeabi-v7a] SharedLibrary  : libijksdl.so
[armeabi-v7a] SharedLibrary  : libijkplayer.so
[armeabi-v7a] Install        : libijkplayer.so => libs/armeabi-v7a/libijkplayer.so
[armeabi-v7a] Install        : libijksdl.so => libs/armeabi-v7a/libijksdl.so
[armeabi-v7a] Install        : libijkutil.so => libs/armeabi-v7a/libijkutil.so
/Users/aven/work/video/ijkplayer/android
Android NDK: ERROR:/Users/aven/work/video/ijkplayer/android/ijkmediaplayer-x86/jni/ffmpeg/Android.mk:ijkffmpeg: LOCAL_SRC_FILES points to a missing file    
Android NDK: Check that /libijkffmpeg.so exists  or that its path is correct   
/Users/aven/Android/android-ndk-r10c/build/core/prebuilt-library.mk:45: *** Android NDK: Aborting    .  Stop.
/Users/aven/work/video/ijkplayer/android到libijkffmpeg.so

{% endhighlight %}

这个时候只能一步一步回溯,首先查看compile-ffmpeg.sh

{% highlight bash lineanchors %}

UNI_BUILD_ROOT=`pwd`
FF_TARGET=$1
set -e
set +x

FF_ALL_ARCHS="armv5 armv7a x86 arm64-v8a"
FF_ACT_ARCHS="armv5 armv7a x86"

echo_archs() {
    echo "===================="
    echo "[*] check archs"
    echo "===================="
    echo "FF_ALL_ARCHS = $FF_ALL_ARCHS"
    echo "FF_ACT_ARCHS = $FF_ACT_ARCHS"
    echo ""
}

case "$FF_TARGET" in
    "")
        echo_archs
        sh tools/do-compile-ffmpeg.sh armv7a
    ;;
    armv5|armv7a|x86|arm64-v8a)
        echo_archs
        sh tools/do-compile-ffmpeg.sh $FF_TARGET
    ;;
    all)
        echo_archs
        for ARCH in $FF_ACT_ARCHS
        do
            sh tools/do-compile-ffmpeg.sh $ARCH
        done
    ;;
    clean)
        echo_archs
        for ARCH in $FF_ALL_ARCHS
        do
            cd ffmpeg-$ARCH && git clean -xdf && cd -
        done
        rm -rf ./build/ffmpeg-*
    ;;
    check)
        echo_archs
    ;;
    *)
        echo "Usage:"
        echo "  compile-ffmpeg.sh armv5|armv7a|x86|arm64-v8a"
        echo "  compile-ffmpeg.sh all"
        echo "  compile-ffmpeg.sh clean"
        echo "  compile-ffmpeg.sh check"
        exit 1
    ;;
esac

echo "\n--------------------"
echo "[*] Finished"
echo "--------------------"
echo "# to continue to build ijkplayer, run script below,"
echo "sh compile-ijk.sh "

{% endhighlight %}

眼尖的话应该看到了，这里也是需要跟参数来决定编译的版本，问题引刃而解。

再次编译成功.