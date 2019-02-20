---
layout: post
title: "Andorid内Aspectj切面失效分析"
description: ""
header_image: "/assets/images/2019-02-20.jpg"
keywords: "Aspectj"
tags: [Aspectj]
---
{% include JB/setup %}

## 背景

通过切面编程，可以做一些源码的bug修复，也可以动态插入模块，最近发现开发期间切面插入的内存泄漏检测失效，本文为排查aop失效的一些采坑记录

## app类查找
既然结果是内存泄漏检测工具不生效，有可能是sdk没集成，也有可可能是切面逻辑没生效。

首先检查构建内是否存在目标代码，检测办法有很多，可以反编译，也可以利用Andorid的构建工具。
我们一apk为输入，检查一下dex内是否存在特定类的定义：


`./findClassDefinition "Lcom/squareup/leakcanary/LeakCanary;"`


```shell
#!/usr/bin/env bash
##################################################################
##
##  find class definition inside apk
##
##
##  Author: Chaobin Wu
##  Email : chaobinwu89@gmail.com
##
#################################################################
die() {
  echo "$*"
  exit 1
}

target=$1
unzip app.apk -d apk
dex=(apk/*.dex)
for file in "${dex[@]}"; do
  if [ -f $file ]; then
    echo $file
    /Users/aven/Android/sdk/build-tools/28.0.3/dexdump -f $file |grep "Class descriptor"|grep $target
  fi
done
```
观察日志，内存泄漏的SDK已经正常集成在apk中，可以缩小排查范围：切面织入是否正常

## 织入排查

这一步主要借助构建日志，如果不知道关键词，需要猜测，比如我们进行切面使用的是aspectj，结合aspect的gralde插件，因此可能从这两块入手

通过源码分析，我们使用的aspect插件会在织入代码的的整个过程中打印日志：

```java
@Override
void transform(Context context
               , Collection<TransformInput> inputs
               , Collection<TransformInput> referencedInputs
               , TransformOutputProvider outputProvider
               , boolean isIncremental) throws IOException, TransformException, InterruptedException {

    def hasAjRt = false
    for (TransformInput transformInput : inputs) {
        for (JarInput jarInput : transformInput.jarInputs) {
            if (jarInput.file.absolutePath.contains(ASPECTJRT)) {
                hasAjRt = true
                break
            }
        }
        if (hasAjRt) break
    }

    //clean
    if (!isIncremental){
        outputProvider.deleteAll()
    }

    if (hasAjRt){
        doAspectTransform(outputProvider, inputs)
    } else {
        println "there is no aspectjrt dependencies in classpath, do nothing "
        inputs.each {TransformInput input ->
            input.directoryInputs.each {DirectoryInput directoryInput->
                def dest = outputProvider.getContentLocation(directoryInput.name,
                        directoryInput.contentTypes, directoryInput.scopes,
                        Format.DIRECTORY)
                FileUtil.copyDir(directoryInput.file, dest)
                println "directoryInput = ${directoryInput.name}"
            }

            input.jarInputs.each {JarInput jarInput->
                def jarName = jarInput.name
                def dest = outputProvider.getContentLocation(jarName,
                        jarInput.contentTypes, jarInput.scopes, Format.JAR)

                FileUtil.copyFile(jarInput.file, dest)
                println "jarInput = ${jarInput.name}"
            }
        }
    }
}

```
通过关键词aspect我们在构建日志中发现了疑点：

```
[exec] All input files are considered out-of-date for incremental task ':xxx:transformClassesWithAspectTransformForWubaRelease'.
 [exec] there is no aspectjrt dependencies in classpath, do nothing 
 [exec] directoryInput = 227d4cde62a03f51cebc924536e7bc58a25234d7
 [exec] jarInput = android.local.jars:umeng-analytics-v6.0.6.jar:2eb55ec9f71567b44cd7ff07aaee798f8e6399fe_7fbda719eb29ecc1fe8606561b0f5e00
 [exec] jarInput = org.aspectj:aspectjrt:1.8.9_7cbb886c3064be16fb8fef6099002200

```
`there is no aspectjrt dependencies in classpath, do nothing`整个提示在插件源码中也可以找到出处，本意是判定当前编译路径中不包含切面所需要的aspectjrt，结合日志和源码，整个错误提示是有问题的，

`[exec] jarInput = org.aspectj:aspectjrt:1.8.9_7cbb886c3064be16fb8fef6099002200`这个输出我们可以知道依赖是存在的。

这里我们为了方便管理过滤源码，提高效率和代码简洁性，使用了`android-aspectjx`

在项目的github主页上，也发现类似返回：

[https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx/issues/50](https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx/issues/50)

总结一下就是1.1.1之前的版本在jdk8环境下切面处理失败，导致编译正常，但是没有成功织入代码，具体修复说明详见：
[https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx/commit/ceab2726c0a574f6df28382eb5e75d7ecfc512ee](https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx/commit/ceab2726c0a574f6df28382eb5e75d7ecfc512ee)

从提交摘要来看，作者将对aspectjrt的判断逻辑调整了了，针对非test的case不在判断。

## 问题修复

修复切面失效的有两个办法：1.升级android-aspeect插件到修改版本；2.不使用该插件，自行配置java编译器参数
方案1可以选择升级到2.x

验证织入是否成功，可以检查编译文件，相比较排查apk文件，更加直观。具体路径为
`build/intermediates/transforms/ajx/ `
一层层递进查找，知道看到一堆的jar文件，这些就是class的归档jar包，通过遍历这些jar找到你的目标插入代码类，就可以肉眼确认切面是否插入成功。
```shell
#!/usr/bin/env bash
##################################################################
##
##  find keyword inside jar
##
##
##  Author: Chaobin Wu
##  Email : chaobinwu89@gmail.com
##
#################################################################
die() {
  echo "$*"
  exit 1
}
echo "start"
target=$1
jar=(xx/build/intermediates/transforms/ajx/wuba/debug/*.jar)
for file in "${jar[@]}"; do
  if [ -f $file ]; then
    echo $file
    unzip -l $file |grep $target
  fi
done
```
## 相关连接

* https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx/blob/v1.0.10/aspectjx/src/main/groovy/com/hujiang/gradle/plugin/android/aspectjx/AspectTransform.groovy
* https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx/issues/50
* https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx/commit/ceab2726c0a574f6df28382eb5e75d7ecfc512ee