---
layout: post
title: "自定义Gradle插件，你所需要知道的一切"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-05-09-01.png
keywords: "gradle plugin"
tags: [Gradle]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-05-09-01.png)


## 前言
这是一篇迟到的笔记，15年已经创建草稿，但是知道今天才真正的动笔，实在惭愧

## 热身知识
一般写gradle插件都是为了构建项目，提供一些小功能，在整个插件开发中Project和Task是两个最重要的概念；  
* Project比较好理解，就是工程主体，他包含一些基本的属性;
* Task是Project内定义的实际干活的对象，比如clean，assembleDebug都是Task;

Gradle插件开发可以使用java，groovy，scala等语言，他们都是基于jvm的语言；

## 写一个Hello world插件
目前来说插件可以直接在build.gradle内实现，也可以独立到单独文件实现，根据官方教程，我们分别看一下两种实现；  

先来看一下直接在build.gradle内的实现：
创建build.gradle文件

    touch build.gradle

用你习惯的任意文本编辑器打开build.gradle，敲入下面的groovy代码

{% highlight groovy %}

/**
* 定义Plugin
*/
class HelloPlugin implements Plugin<Project> {

    void apply(Project project) {
        project.extensions.create("author", Author)
        project.task('echo') << {
            println 'Author information:'
            println project.author.name
            println project.author.email

        }
    }
}
/**
* 定义model
*/
class Author {
    def name
    def email
}

/**
* 使用Plugin
*/
apply plugin: HelloPlugin

/**
* 为当前project初始化author的成员
/
author {
    name = "Chaobin Wu"
    email = "wuchaobin@58ganji.com"
}

{% endhighlight %}


这段代码实际上实现了Plugin,自定义了名为GreetingPlugin的插件，同时定义了一个名为Author的model，在Plugin中我们声明了名为echo的task，并附加了一段操作,执行一段输出操作；  
    
现在我们在CLI内执行echo这个task：

    gradle echo

对应的输出如下：  

{% highlight bash %}
avens-MacBook-Pro:hello-world aven$ gradle echo
:echo
Author information:
Chaobin Wu
wuchaobin@58ganji.com
{% endhighlight %}
实际上也可以继承DefaultTask来定义Task，这和java中定义一个class是一样的，然后利用@TaskAction注解标记那个方法为task执行时的入口

{% highlight groovy %}
public class DoJob extends DefaultTask {
    def value

    @TaskAction
    def echo() {
        println(value)
    }
}
task dojob(type: net.avenwu.gradle.DoJob) {
    value = 'Do something'
}
{% endhighlight %}

以后DoJob可以作为一个Task的具体类型派生出各种task，比如task dojob，注意指定type必须包含包名，这和java一样，否则会找不到类；

一个最基本的gradle插件实际上就完成了，但是如果我们需要发布这个插件，并在其他工程中引用要怎么做呢？  
下面我们把插件实现剥离出来，放入一个标准的独立的工程内，这样后续我们就可以对这个插件进行发布操作： 

### build.gradle
gradle插件的编译脚本也是gradle，所以我们任然需要一个build.gradle，但是这次我们只在里面写编译的配置信息：

{% highlight groovy %}

apply plugin: 'groovy'

dependencies {
    compile gradleApi()
    compile localGroovy()
}
{% endhighlight %}
gradle插件我们是用groovy写的，因此需要引用groovy plugin，然后是必须的一些依赖；

### property配置
为了让编译系统能找到我们插件的实现类入口，我们需要建立一个配置文件：

    src/main/resources/META-INF/gradle-plugins/net.avenwu.gradle.HelloPlugin.properties

内容就是一个键值对，值是我们的实现类，包括包名
    
    implementation-class=net.avenwu.gradle.HelloPlugin

properties的命名是plugin的group id,内容中的键值对的value则是对应plugin的源码实现类；


### 发布插件
发布一般来说我们会发到一个maven仓库，比如开源的jcenter，maven,也可以是公司内部搭建的maven库，这里我们直接发布到本地目录

{% highlight groovy %}
group = 'net.avenwu.gradle'
version = '1.0-SNAPSHOT'

uploadArchives {
    repositories {
        mavenDeployer {
            repository(url: uri('../repo'))
        }
    }
}
{% endhighlight %}

执行uploadArchives发布插件

    gradle uploadArchives

{% highlight groovy %}

avens-MacBook-Pro:TestPlugin aven$ gradle uploadArchive
:compileJava UP-TO-DATE
:compileGroovy
:processResources UP-TO-DATE
:classes
:jar
:uploadArchives

BUILD SUCCESSFUL

Total time: 9.443 secs

This build could be faster, please consider using the Gradle Daemon: https://docs.gradle.org/2.7/userguide/gradle_daemon.html
{% endhighlight %}

### 引用插件
现在本地的repo目录下已经有了发布好的插件: 

{% highlight bash %}
avens-MacBook-Pro:TestPlugin aven$ pwd
/Users/aven/work/gradle-demo/repo/net/avenwu/gradle/TestPlugin
avens-MacBook-Pro:TestPlugin aven$ ls -al
total 24
drwxr-xr-x   6 aven  staff   204 May  9 22:36 .
drwxr-xr-x   3 aven  staff   102 May  9 22:36 ..
drwxr-xr-x  41 aven  staff  1394 May  9 23:07 1.0-SNAPSHOT
-rw-r--r--   1 aven  staff   285 May  9 23:07 maven-metadata.xml
-rw-r--r--   1 aven  staff    32 May  9 23:07 maven-metadata.xml.md5
-rw-r--r--   1 aven  staff    40 May  9 23:07 maven-metadata.xml.sha1
{% endhighlight %}

我们可以在建立一个测试项目来引用这个插件,测试工程简单起见可以只包含一个gradle脚本
{% highlight groovy %}

buildscript {
    repositories {
        maven {
            url uri('../repo')
        }
    }
    dependencies {
        classpath group: 'net.avenwu.gradle', name: 'TestPlugin',
                  version: '1.0-SNAPSHOT'
    }
}
apply plugin: 'net.avenwu.gradle.HelloPlugin'

author {
    name = "Chaobin Wu"
    email = "wuchaobin@58ganji.com"
}

task dojob(type: net.avenwu.gradle.DoJob) {
    value = 'Do something'
}
{% endhighlight %}

现在在CLI内执行gradle task会有发现有两个我们自定义的task,注意观察Other tasks部分：  

{% highlight bash %}
avens-MacBook-Pro:hello-world aven$ gradle task
:tasks

------------------------------------------------------------
All tasks runnable from root project
------------------------------------------------------------

Help tasks
----------

Other tasks
-----------
dojob
echo

BUILD SUCCESSFUL

Total time: 4.796 secs

This build could be faster, please consider using the Gradle Daemon: https://docs.gradle.org/2.7/userguide/gradle_daemon.html
{% endhighlight %}

## 小结
遵循gradle插件开发的基本步骤可以很快写出一个hello world，但是插件总归是要有意义的，因此需要对gradle进行更完整的学习以便掌握更过api，方能在自定义插件时得心应手。

为了方便起见涉及的源码已打包，可以在这里获取：[gradel-demo.tar.gz]({{ site.baseurl }}/assets/files/gradle-demo.tar.gz)

## 参考
* [https://docs.gradle.org/current/userguide/custom_plugins.html](https://docs.gradle.org/current/userguide/custom_plugins.html)
* [https://plugins.gradle.org/docs/publish-plugin](https://plugins.gradle.org/docs/publish-plugin)