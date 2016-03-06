---
layout: post
title: "RxJava/Retrolambda with Android"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-15.jpg
description: ""
category: 
tags: []
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-15.jpg)

## 前言

Android开发领域出现了越来越多的框架，而一款便于开发的框架很快就能吸引一大批粉丝关注，比如本文介绍的[RxJava](https://github.com/ReactiveX/RxJava)，[RxAndroid](https://github.com/ReactiveX/RxAndroid)

下面简单介绍下怎么在项目中集成RxAndroid与相关配置。

## RxAndroid
配置RxAndroid本身并不麻烦只是一句话，添加依赖包即可，打开build.gradle：  
compile 'io.reactivex:rxandroid:0.24.0'

但是为了使RxJava书写更便捷我们还需要Retrolambd支持Java8的lambda语法；

## Retrolambda
orfjackal开发的retrolambd,在AndroidStudio中使用的话，可以配合evant写的gradle plugin：  

同样在build.gradle中添加即可，
	
	apply plugin: 'me.tatarka.retrolambda'  
   
为了使用lambda需要下载jdk8,并且在AndroidStudio中配置其目录，Android Studio -> Preferences -> Path Variables and adding

	JAVA_8 /Library/Java/JavaVirtualMachines/jdk1.8.0_25.jdk/Contents/Home

最后在build.gralde中再加上编译版本

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }

完整的build.gradle配置如下:

{% highlight groovy %}
apply plugin: 'com.android.application'
apply plugin: 'me.tatarka.retrolambda'

android {
    compileSdkVersion 22
    buildToolsVersion "23.0.0 rc2"

    defaultConfig {
        applicationId "avenwu.net.rxdemo"
        minSdkVersion 15
        targetSdkVersion 22
        versionCode 1
        versionName "1.0"
    }
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile 'com.android.support:appcompat-v7:22.2.0'
    compile 'io.reactivex:rxandroid:0.24.0'
}
{% endhighlight %}


## 编写代码

如果没有以外的话，即可以开始写代码:

{% highlight java %}
public class MainActivity extends AppCompatActivity {
    TextView mLabel;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mLabel = (TextView) findViewById(R.id.text);
        String[] cities = {"Beijing", "Hangzhou", "Nanjing", "Nanchang"};
        Observable.from(cities)
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(city -> {
                    Log.d("Test", city);
                    mLabel.append(city + " ");
                });
    }
}
{% endhighlight %}
编译并运行

![截图](http://7u2jir.com1.z0.glb.clouddn.com/device-2015-06-25-093441.png)

## 参考
* [https://github.com/ReactiveX/RxJava](https://github.com/ReactiveX/RxJava)
* [https://github.com/orfjackal/retrolambda](https://github.com/orfjackal/retrolambda)
* [https://github.com/evant/gradle-retrolambda](https://github.com/evant/gradle-retrolambda)