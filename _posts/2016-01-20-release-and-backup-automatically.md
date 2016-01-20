---
layout: post
title: "Android发版、备份自动化之路"
description: "elease and backup automaticall"
category: 
tags: [android]
---
{% include JB/setup %}

	* 自动化脚本在项目开发和管理中非常重要，不但可以简化工作也更加安全，不会像人工操作遗漏步骤
	* 在日常管理项目和版本迭代中，为了方便，根据公司项目的现状，笔者陆续建立和完善了相应的打包备份脚本，经过历次迭代，效果还不错；

---

##前言
通过脚本解决了什么问题？

* 首先自然是打包，生成发行用的apk文件；
* 签名验证，确认生成的包是符合发行的签名；
* 资源压缩，如css，js文件；
* 记录发版日志，比如当前打包的git摘要，时间，作者，版本信息；
* 自动安装，根据有无设备连接，自动安装至设备；

##逐一实现目标

过去打包，很多时候用的ant，自gradle在android领域内兴起的这两年，脚本也慢慢都转向了支持gradle；

	    ./gradlew clean assemble"$flavor"Release

这里根据输入的flavor进行release；

得到签名文件后，可以对其做签名验证

	    signature=`unzip -p build/outputs/apk/android-$flavor-armeabi-release.apk META-INF/CERT.RSA | keytool -printcert | grep MD5`
	    checkfail "Release failed: flavor=$flavor"
	    
解压安装包，读取签名摘要，和正式签名文件的MD5值作比较；

接下来将验证通过的apk和mapping信息备份；

	echoInfo "Copy file to backup"
	folderName="v$version"
	mkdir release/$folderName
	cp -rf build/outputs/apk/android-$flavor-armeabi-release.apk release/$folderName/
	cp -rf build/outputs/mapping release/$folderName/
	echoShipLog $flavor $version
	
最后可以适当清理一下打包过程中产生的垃圾文件，部署apk至android设备

	cleanTemp(){
    	echoInfo "Cleaning temporary files..."
    	git checkout assets/md/* tools/geek.keystore tools/geek.properties
	}
	
	launchApp(){
    	adb shell am start -n $LAUNCH_PAGE -a android.intent.action.MAIN -c android.intent.category.LAUNCHER
	}

##效果
各个目标点都实现后，整体串联起来；加上细节判断，下面完整的脚本代码；

{% highlight bash %}
#!/usr/bin/env bash

#############
# constant
#############

LAUNCH_PAGE="com.jikexueyuan.geekacademy/com.jikexueyuan.geekacademy.ui.activity.ActivitySplash"
CERTIFICATE_MD5="你的签名MD5摘要信息"
LOG_FILE="release/shipLog.md"
DEVICE="none"

RED_COLOR='\033[1;31m'
GREEN_COLOR='\033[1;32m'
RES='\033[0m'

#############
# functions
#############

launchApp(){
    adb shell am start -n $LAUNCH_PAGE -a android.intent.action.MAIN -c android.intent.category.LAUNCHER
}

checkfail(){
    if [ ! $? -eq 0 ];then
        echoWarn "$1"
        exit 1
    fi
}
echoWarn(){
    echo -e  "${RED_COLOR}$1${RES}"
}
echoInfo(){
    echo -e  "${GREEN_COLOR}$1${RES}"
}

checkDevices(){
    firstDevice="`adb devices | sed -n 2p`"
    for d in ${firstDevice} ; do
        echoInfo "Found device: $d"
        DEVICE="$d"
        return
    done
}

#Usage help
echoHelp(){
    echoInfo "##This script is used to ship release apk with specify workflow"
    echo "Usage:"
    echoInfo "-v version_number"
    echo -e "\\tRelease default flavor(Jikexueyuan)and backup with version_number"
    echoInfo "-f flavorName"
    echo -e "\\tRelease the supplied flavor by -f"
    echoInfo "-h"
    echo -e "\\tShow the help information"
    echoInfo "-c"
    echo -e "\\tClear all script generated temp files"
    echoInfo "Example"
    echo -e "\\tdeploy -v 4.0.0 -f Jikexueyuan -c"
    echo -e "\\tdeploy -v 4.0.0 -c"
}

prepareForShip(){
    if [ -f tools/release.keystore ]; then
        echoInfo "Prepare release keystore..."
        cd tools/
        cp release.keystore geek.keystore
        cp release.properties geek.properties
        cd ..
    fi

    ./tools/compressScript
    ./tools/updateApi

    checkDevices
    if [ $DEVICE != "none" ]; then
        adb uninstall com.jikexueyuan.geekacademy
    fi
}

backup(){
    echoInfo "Backup released flavor..."
    apk="build/outputs/apk/android-*-release.apk"
    ./tools/backup $apk $version
}

assembleFlavor() {
    flavor=$1
    version=$2
    echoInfo "Start releasing, flavor=$flavor..."

    checkDevices
    if [ $DEVICE != "none" ]; then
        ./gradlew clean assemble"$flavor"Release install"$flavor"Release
    else
        ./gradlew clean assemble"$flavor"Release
    fi
    checkfail "assemble failed for flavor:$1"

    flavor=`echo $flavor | tr '[:upper:]' '[:lower:]'`
    echoInfo "Check signature of $flavor..."
    signature=`unzip -p build/outputs/apk/android-$flavor-armeabi-release.apk META-INF/CERT.RSA | keytool -printcert | grep MD5`
    checkfail "Release failed: flavor=$flavor"

    if [[ $signature == *"$CERTIFICATE_MD5"* ]];then
        echoInfo "Signature verified";
        if [ $version != "undefined" ]; then
            echoInfo "Copy file to backup"
            folderName="v$version"
            mkdir release/$folderName
            cp -rf build/outputs/apk/android-$flavor-armeabi-release.apk release/$folderName/
            cp -rf build/outputs/mapping release/$folderName/

            echoShipLog $flavor $version
        fi
    else
        echoWarn "Signed with debug key"
    fi
}

echoShipLog(){
    echoInfo "Shipping completed, write log to $LOG_FILE"
    timestamp=`date +%Y-%m-%d/%r`
    revision=`git rev-parse --short HEAD`
    branch=`git br |grep "*"`
    author=`whoami`
    echo "##$timestamp" >> $LOG_FILE
    echo "flavor=$1" >> $LOG_FILE
    echo "version=$2" >> $LOG_FILE
    echo "revision=$revision" >> $LOG_FILE
    echo "branch=$branch" >> $LOG_FILE
    echo "author=$author" >> $LOG_FILE
}

shipNow(){
    set -e
    flavor=$2
    version=$1

    prepareForShip
    assembleFlavor $flavor $version

    checkDevices
    if [ $DEVICE != "none" ]; then
        launchApp
    fi
}

cleanTemp(){
    echoInfo "Cleaning temporary files..."
    git checkout assets/md/* tools/geek.keystore tools/geek.properties
}

###################
# Release
###################

flavorName="Jikexueyuan"
version="undefined"
clean=0

while getopts "v:f:hc" arg
do
    case $arg in
         v)
            version="$OPTARG"
            ;;
         f)
            flavorName="$OPTARG"
            ;;
         h)
            echoHelp
            exit 1
            ;;
         c)
            clean=1
            ;;
         ?)
            echoWarn "Unknown arguments"
            echoHelp
            exit 1
            ;;
    esac
done

echoInfo "version=$version,flavor=$flavorName"
shipNow $version $flavorName

if [ $clean -eq 1 ]; then
    cleanTemp
fi

{% endhighlight %}

---

