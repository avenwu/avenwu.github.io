---
layout: post
title: "Kotlin with Android"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-44.jpg
description: "Kotlin开发发Android应用程式"
category: 
tags: [kotlin]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-44.jpg)

## 前言
偶然发现一门开发Android的新语言：[kotlin](http://kotlinlang.org/docs/reference/coding-conventions.html)  
和java一样也是是基于jvm的语言，并且在很多细节方面比java书写起来可以减少不少的荣誉代码，比如lambdas；
用java开发想使用java8的lambdas需要借助其他的工具比如retrolambda。

## 环境搭建
如果已经安装了常规的Android开发环境，配合AndroidStudio，只需要添加kotlin的plugin即可，具体安装步骤参考：

[http://kotlinlang.org/docs/tutorials/kotlin-android.html](http://kotlinlang.org/docs/tutorials/kotlin-android.html)

[http://kotlinlang.org/docs/tutorials/android-plugin.html](http://kotlinlang.org/docs/tutorials/android-plugin.html)

{% highlight groovy %}
apply plugin: 'com.android.application'
apply plugin: 'kotlin-android'

android {
    compileSdkVersion 22
    buildToolsVersion "23.0.0 rc2"

    defaultConfig {
        applicationId "avenwu.net.kotlinandroid"
        minSdkVersion 15
        targetSdkVersion 22
        versionCode 1
        versionName "1.0"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
    sourceSets {
        main.java.srcDirs += 'src/main/kotlin'
    }
}

dependencies {
    compile fileTree(include: ['*.jar'], dir: 'libs')
    compile 'com.android.support:appcompat-v7:22.2.0'
    compile "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"
}
buildscript {
    ext.kotlin_version = '0.12.613'
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
        classpath "org.jetbrains.kotlin:kotlin-android-extensions:$kotlin_version"
    }
}
repositories {
    mavenCentral()
}

{% endhighlight%}

## Kotlin编写简单的Android应用
{% highlight java %}

package avenwu.net.kotlinandroid

import android.support.v7.app.AppCompatActivity
import android.os.Bundle
import android.view.Menu
import android.view.MenuItem
import kotlinx.android.synthetic.activity_main.*

public class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        hello.setText("What the amazing Kotlin is!")
    }

    override fun onCreateOptionsMenu(menu: Menu?): Boolean {
        getMenuInflater().inflate(R.menu.menu_main, menu)
        return true
    }

    override fun onOptionsItemSelected(item: MenuItem?): Boolean {
        val id = item!!.getItemId()
        if (id == R.id.action_settings) {
            return true
        }

        return super.onOptionsItemSelected(item)
    }
}

{% endhighlight %}


类似于ButterKnife,通过org.jetbrains.kotlin:kotlin-android-extensions插件
layout内的控件可以实现自动绑定import kotlinx.android.synthetic.activity_main.*  
这里的activity_main是对应的xml布局

## Kotlin语法简单介绍
总体感觉kotlin和swift，js等都非常相似；

{% highlight kotlin %}
/**
 * Created by aven on 6/24/15.
 */
fun main(args: Array<String>) {
    println("Hello World")
    println("sum=" + add(5, 8));
    println("sum=" + add2(5, 8));
    println("sum=" + add3(5, 8));
    saySomthing("Haaa")
    saySomthing(12)
    checkValue(121)
    checkValue("HelloKotlin")
    checkValue("sakjnx")
    rangTest()
    showName(listOf("X Man", "Bitch", "Monkey", "ZYX"));
    var ss: String?
    ss = null
    ss?.let { println("$ss") }

}

/**
 * 定义方法
 */
fun add(number1: Int, number2: Int): Int {
    return number1 + number2;
}

/**
 * 表达式写法
 */
fun add2(number1: Int, number2: Int): Int = number1 + number2

/**
 * 类型推理
 */
fun add3(number1: Int, number2: Int) = number1 + number2

/**
 * 自动类型转换
 */
fun saySomthing(words: Any) {
    if (words is String) {
        //if语句内自动转换为String
        var length = words.length();
        println("${words} contains ${length} characters")
        return
    }
    println("${words} is not String")
}

/**
 * when语句，类似switch
 */
fun checkValue(value: Any) {
    when (value) {
        is Int -> println("${value} is Int")
        "HelloKotlin" -> println("${value} is String")
        else -> println("I don't care about ${value}")
    }
}

/**
 * Lambdas
 */
fun showName(names: List<String>) {
    names filter { it.contains("X") } map { it.toLowerCase() } forEach { println("${it}") }
}

fun rangTest() {
    for (i in 1..20) {
        println("$i")
    }
}

data class Customer(val name: String, val email: String)
{% endhighlight %}
## 参考

[kotlinlang.org](http://kotlinlang.org/docs/reference/coding-conventions.html)