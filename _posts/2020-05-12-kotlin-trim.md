---
layout: post
title: "瘦身|Kotlin与Version资源在App中的分析"
description: "Kotlin瘦身"
header_image: /assets/img/2020-05-12-01.png
keywords: "Kotlin"
tags: [Kotlin]
---
{% include JB/setup %}
![img](/assets/img/2020-05-12-01.png)

* 目录
{:toc #markdown-toc}


不知从何时起，发现APK中陌生文件越来越多。比如本文分析的kotlin相关文件和version配置信息

```
META-INF/kotlin-stdlib.kotlin_module
kotlin/time/TimedValue.kotlin_metadata
kotlin/kotlin.kotlin_builtins
META-INF/android.support.design_material.version
META-INF/androidx.appcompat_appcompat.version
```

这些文件根据目录和后缀可以分文四种

- .kotlin_module 位于META-INF下
- .kotlin_metadata 位于kotlin下
- .kotlin_builtins 位于kotlin下
- .version文件 位于META-INF下

为了搞清楚这些文件是不是可以被去除优化，需要做研究一下他们的作用。

## 资源分布情况

使用kotlin和android的support后，这些文件就会存在，我们看下市场上一些App是否存在这些资源

| App      | .kotlin_module/数量 | .kotlin_metadata/数量 | .kotlin_builtins/数量 | .version/数量 |
| -------- | ------------------- | --------------------- | --------------------- | ------------- |
| 京东     | 存在/3              | 不存在                | 存在/6                | 存在/87       |
| 今日头条 | 存在/193            | 存在/275              | 存在/7                | 存在/39       |
| QQ       | 不存在              | 不存在                | 不存在                | 不存在        |
| 快手     | 不存在              | 不存在                | 不存在                | 不存在        |
| 微信     | 不存在              | 不存在                | 不存在                | 不存在        |
| 美团     | 不存在              | 不存在                | 不存在                | 存在/6        |
| 58同城   | 存在/38             | 存在/275              | 存在/7                | 存在/43       |

分析特征文件是否存在，这里简单采用了文件匹配
```
unzip -l app.apk |grep version$|wc -l
```
根据数据情况，可以知道

- QQ，快手，微信，美团，都没有kotlin相关的资源，莫非是没有使用kotlin开发？
- 头条和同城的数据接近，都采用了kotlin开发，京东有少量的kotlin模块资源
- .version不存在的App疑似做了文件删除

另外再看一组数据，通过AS创建一个启用Kotlin的Demo工程，其构建的APK大致如下:

| App  | kotlin_module/数量 | kotlin_metadata/数量 | kotlin_builtins/数量 | .version/数量 |
| ---- | ------------------ | -------------------- | -------------------- | ------------- |
| Demo | 存在/20            | 存在/275             | 存在/7               | 存在/44       |

kotlin和version的大小达到了`3.26%`，当然随着业务模块增大，这个比重应该会下降。

> 97.7+86.1-40.8-39.1-1=102.9KB

![sample](/assets/images/sample.png)

当对Demo剔除这些文件后，Demo任然可以正常运行，页面功能表现正常。

对比数据后，apk大小减少比重超过了`3.26%`，整体Debug安装包从3.4MB减小到3.2MB，对比减小的部分可以发现还有`MANIFEST.MF`，从39.1KB变为25.8KB。

> 由于apk内文件减少，因此清单文件的行数也相应减少，从而减小了MANIFEST.MF文件大小。

![sample-diff](/assets/images/sample-diff.png)

## kotlin_module文件

二进制文件, 与kotlin的作用域相关。

> .kotlin_module文件的最初目的是将包部件映射存储到已编译的类文件中.编译器使用它们来解析附加到项目的库中的顶级函数调用.此外,Kotlin反射使用.kotlin_module文件在运行时构造顶级成员元数据。

上面这句话有些饶舌，本质上是为了优化顶级函数/变量定义时潜在的包名冲突，通过独立的kotlin_module实现快速查找。另一个是反射使用。

例如在kotlin中可以书写如下代码，不指定类而直接创建一个包名顶级的函数。

```kotlin
package foo.bar
fun demo() { ... }
```

编译后，与Java中如下写法异曲同工
```kotlin
package foo.bar;
public class BarPackage {
    public static void demo() { ... }
}
```

也可以通过注解指定生成类的名字

```kotlin
@file: JvmName("Utils")
package cn.hacktons.trim
fun hello() {}
```

转后生成Utils类

![kotlin-jvm-class](/assets/images/kotlin-jvm-class.png)

针对每个lib/module生成kotlin_module文件，该文件的维度是以lib库为粒度，并不是一个类一个文件

![kotlin-module](/assets/images/kotlin-module.png)

如果再创建一个libkotlin，则kotlin_module会再增加一个

![kotlin-module-2](/assets/images/kotlin-module-2.png)

除了开发者的kotlin模块新增的文件，其他都是kotlin库产生的。

> 从这个角度来说，多仓库混合的大项工程，似乎要慎重对待这些文件，不要删除？

在Demo测试中，删除并没有产生任何问题。这个配置是作用于编译期，告诉编译器如何正确生成class代码的，运行期间并不会使用到，APK导包过程时可以删除。如果有使用kotlin reflect可能会出错。

## kotlin_metadata文件

二进制文件，从反编译数据看，kotlin类经过编译后，带上了一个MetaData的注解，并且配置信息非常多，这也直接导致kotlin编写的代码生成的class文件，比java编写的代码生成的class要大。

```java
@Metadata(bv = {1, 0, 3}, d1 = {"\u0000\u0018\n\u0002\u0018\u0002\n\u0002\u0018\u0002\n\u0002\b\u0002\n\u0002\u0010\u0002\n\u0000\n\u0002\u0018\u0002\n\u0000\u0018\u00002\u00020\u0001B\u0005¢\u0006\u0002\u0010\u0002J\u0012\u0010\u0003\u001a\u00020\u00042\b\u0010\u0005\u001a\u0004\u0018\u00010\u0006H\u0014¨\u0006\u0007"}, d2 = {"Lcn/hacktons/trim/MainActivity;", "Landroidx/appcompat/app/AppCompatActivity;", "()V", "onCreate", "", "savedInstanceState", "Landroid/os/Bundle;", "app_debug"}, k = 1, mv = {1, 1, 16})
/* compiled from: MainActivity.kt */
/* renamed from: cn.hacktons.trim.MainActivity */
public final class MainActivity extends AppCompatActivity {
    private HashMap _$_findViewCache;

    public void _$_clearFindViewByIdCache() {
        HashMap hashMap = this._$_findViewCache;
        if (hashMap != null) {
            hashMap.clear();
        }
    }
// ...
}
```

可以看到kt编译后会生成注解，注解中又包含了类名，方法，参数等详细签名情况；这个配置并不会跟随混淆更新。测试过程中发现，不是所有kotlin生成的类都会携带这个注解，比如R文件，BuildConfig不会携带，简单创建一个对象后是会携带所从属的module信息：

```kotlin
package cn.hacktons.trim

class b {
}
```

编译后class

```java
package p008cn.hacktons.trim;

import kotlin.Metadata;

@Metadata(bv = {1, 0, 3}, d1 = {"\u0000\f\n\u0002\u0018\u0002\n\u0002\u0010\u0000\n\u0002\b\u0002\u0018\u00002\u00020\u0001B\u0005¢\u0006\u0002\u0010\u0002¨\u0006\u0003"}, d2 = {"Lcn/hacktons/trim/b;", "", "()V", "app_debug"}, k = 1, mv = {1, 1, 16})
/* compiled from: b.kt */
/* renamed from: cn.hacktons.trim.b */
public final class C1083b {
}
```

这些元数据文件，在Demo中被删除后程序正常运行，说明运行期间也没有用上，但是不排除其他反射调用会涉及。

## kotlin_builtins文件

二进制文件，kotlin基础库的所支持的类库新，如果删除，反射实例化运行时就会直接报错。

![kotlin-reflect-error](/assets/images/kotlin-reflect-error.png)

## .verison文件

纯文本文件，内部一般是一个版本号

```
cat META-INF/androidx.lifecycle_lifecycle-livedata.version 
2.2.0
```

## 微信、QQ、快手与Kotlin？

前面统计时我们知道，有几个App中不包含Kotlin生成的一些默认配置，因此要么是其做了删除，要么是没有使用kotlin。反编译验证一下。

**微信**有使用kotlin

![kotlin-wecaht](/assets/images/kotlin-wecaht.png)

同时可以看到其他一些信息，

* 微信分包用了进程吊起方案

> <service android:name="com.tencent.p105mm.splash.DexOptService" android:process=":dexopt"/>

* 当我们还在艰难的从armeabi升级到v7a时，微信已经在v8a上畅游了
* Flutter也有在使用😀，大厂的速度与魄力
* tinker自然不在话下

---

**QQ**有使用kotlin

![kotlin-qq](/assets/images/kotlin-qq.png)

* qq任然使用armeabi架构
* Flutter也有在使用😀
* 热修复用的自有Qfix

---

**快手**有使用kotlin

![kotlin-kuaishou](/assets/images/kotlin-kuaishou.png)

* 快手包名比较有意思，叫com.smile.gifmaker😀
* 快手使用v7a架构
* Flutter也有在使用😀
* apk中有些不必要的配置，比如proguard文件，appt/AndroidMainfest.xml
* 热修复使用了基于tinker的com.kwai.hotfix

## 小结

默认情况下引入kotlin后包会增大，同时随着业务迭代，产生的配置文件也较多，如果避免使用反射能力，则可以剔除相关文件，也可以有一定瘦身效果。已经注意到并落地的App如微信，QQ，快手。

## 参考

* [Compressing the APK, trying to keep it working / Habr](https://www.techort.com/compressing-the-apk-trying-to-keep-it-working-habr/)
* [What are .kotlin_builtins files and can I omit them from my uberjars?](https://stackoverflow.com/questions/41052868/what-are-kotlin-builtins-files-and-can-i-omit-them-from-my-uberjars)
* [Improving Java Interop: Top-Level Functions and Properties](https://blog.jetbrains.com/kotlin/2015/06/improving-java-interop-top-level-functions-and-properties/)
* [关于 `META-INF/library_release.kotlin_module`](https://deskid.github.io/2019/03/05/about-kotlin-module/)

