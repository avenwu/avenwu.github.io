---
layout: post
title: "Proguard不正确使用===没用"
keywords: "Proguard 优化"
description: "proguard optimize"
category: 
tags: [proguard]
---
{% include JB/setup %}

	商业应用生成发行包时往往会做一些压缩，混淆的保护，android是基于java语言开发的，使用的是java的混淆工具Proguard；
	但是用了Proguard就真的混淆了么？

---

## 前言

最近在反编译公司的项目，发现暴露的信息比较多，于是有了，此文，能不能在减少一些；  
先来看看目前常见的问题：
	
	* model没混淆，出于序列化和反序列化的原因，model很多时候被keep了，包括内部成员和自身类名；
	* 混淆的类，通过某些工具还是可以看到“真身”的影子；
	* 三方库基本维持原样，大量为未混淆，为臃肿的安装包作了贡献；

## 问题分析

1. model混淆问题

model一般来说是可以混淆的，如果用到了，写死的反射，或者用了依赖于反射，命名依赖的库，那么就需要避免混淆；

那么怎么keep住？相信很多人会和笔者一样，一拍脑门，直接keep全部
	
	-kepp class com.your_package_name.modle.** { *; }

当然这样是可以的，但是，keep太强了，可以根据情况适当优化，比如，类名是否可以混淆；

	-keepclassmembers class com.your_package_name.model.** { *; }

keepclassmembers可以保留成员方法和属性，但是类名还是可以混淆的；

2. 为什么混淆的类，还会暴露真实的信息

举个例子，下面是用jadx工具反编译后得到的样本；

{% highlight java %}
	/* compiled from: FlavorKit */
public class a {
    public static void a(Context context) {
        if (!org.videolan.a.c.equals("debug")) {
        }
    }
}
{% endhighlight %}

很好奇混淆的代码上面的注释里面居然出现了真实信息，简直不可原谅。

为了找出原因，我们去jadx的源码找一下，这段注释是怎么生成的

	git clone https://github.com/skylot/jadx
	cd jadx
	grep -r "compiled from:" ./
	.//jadx-core/src/main/java/jadx/core/codegen/ClassGen.java:			code.startLine("/* compiled from: ").add(sourceFileAttr.getFileName()).add(" */");
	
grep很快就找到了这段话的出处，定位到ClassGen.java，可以返现下面两个方法，也就是说类名信息是直接冲ClassNode中获取的；

{% highlight java %}
	private void insertSourceFileInfo(CodeWriter code, AttrNode node) {
		SourceFileAttr sourceFileAttr = node.get(AType.SOURCE_FILE);
		if (sourceFileAttr != null) {
			code.startLine("/* compiled from: ").add(sourceFileAttr.getFileName()).add(" */");
		}
	}

	private void insertRenameInfo(CodeWriter code, ClassNode cls) {
		ClassInfo classInfo = cls.getClassInfo();
		if (classInfo.isRenamed()
				&& !cls.getShortName().equals(cls.getAlias().getShortName())) {
			code.startLine("/* renamed from: ").add(classInfo.getFullName()).add(" */");
		}
	}
{% endhighlight %}

换句话说，混淆后的字节码中保留的这些文件信息。直觉告诉我们，是不是混淆配置出了问题？

通读配置文件，发现umeng的配置配置中有这么两句：

	-keepattributes SourceFile,LineNumberTable
	
真相大白了，SourceFile根据Proguard的手册，表示的是java文件的各种文件属性，如果这个被keep了，当然反编译工具就可以读取到相关信息,我们直接去除他；

	-keepattributes LineNumberTable
	
3. 三方未混淆，很臃肿

依赖库的问题，需要具体问题具体分析，不能一刀切的全部keep，比如umeng sdk里面粗暴的吧com.facebook全部keep了。可想而知，如果工程内用了什么facebook出品的库那就和混淆无缘了,比如Fresco；

由于工程内并没有用到facebook分享，可以直接取消keep，而fresco得配置已单独写过了
	
	#-keep public interface com.facebook.**
	
	-keep,allowobfuscation @interface com.facebook.common.internal.DoNotStrip

	# Do not strip any method/class that is annotated with @DoNotStrip
	-keep @com.facebook.common.internal.DoNotStrip class *
	-keepclassmembers class * {
    	@com.facebook.common.internal.DoNotStrip *;
	}
	-dontwarn okio.**
	-dontwarn javax.annotation.**
	
## 检验效果

处理完上述三个问题后，看看安装包是否减小了；
	
	ls -al normal.apk
	-rw-r--r--  1 aven  staff  16734081 Jan 21 10:01 normal.apk
	ls -al optimized.apk
	-rw-r--r--  1 aven  staff  16671065 Jan 25 16:44 optimized.apk

安装包减小了61KB(16734081-16671065=63016);

当然运行也得确保正常；

## 小结

在使用Proguard的时候需要特别小心，有必要的话多核对手册，避免引入某些不必要的参数，导致混淆被"脱裤".

---
	
