---
layout: post
title: "如何正确擦除调试日志"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2018-07-19-01.png
keywords: ""
tags: [proguard]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2018-07-19-01.png)

## 背景

开发期间我们经常会使用android.os.Log进行日志输出，这在调试开发中没有问题，但是如果要在线上包中去除这些日志，就会遇到一些问题。

## 擦除不生效

我们都知道利用Proguard可以做代码混淆，利用`-assumenosideeffects`可以配置需要移除的日志；

```java
public class CustomApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        Logger.info("TestLog", "AAA");
        Logger.info("BBB","CCC", "DDD");
    }
}
public class Logger {
    private final static String TAG = "XXLog";

    public static void info(String msg, Object... args) {
        if (BuildConfig.DEBUG) {
            Log.i(TAG, String.format(msg, args));
        }
    }
}
```

现在我们通过配置来擦除这些日志：

```
# Disable Android logging
-assumenosideeffects class android.util.Log {
  public static boolean isLoggable(java.lang.String, int);
  public static *** v(...);
  public static *** d(...);
  public static *** i(...);
  public static *** w(...);
  public static *** e(...);
}
# custom logger
-assumenosideeffects class cn.hacktons.xx.Logger {
    *;
}
```

但是这样配置确实能生效么？我们可以反编译一下代码：

```java
public class CustomApplication extends Application {
    public void onCreate() {
        super.onCreate();
        C0979c.m3936a("TestLog", (Object) "AAA");
        C0979c.m3937a("BBB", "CCC", "DDD");
    }
}

public class C0979c {
    /* renamed from: a */
    public static void m3938a(String str, Object... objArr) {
    }
}
```

可以发现Logger内部逻辑已经被优化了一些，但是日志调用仍然存在，这并没有达到我们的预期，

> 我们希望连整个日志调用的代码行都擦除掉。

出现这个现象是由于proguard配置中有其他选项阻止了字节码的优化：`-dontoptimize`

确保你的配置中没有这个选项。

## 擦除不彻底

现在我们再次打包并反编译查看下日志情况：

```java
public class CustomApplication extends Application {
    public void onCreate() {
        super.onCreate();
        new Object[1][0] = "AAA";
        Object[] objArr = new Object[]{"CCC", "DDD"};
    }
}
```
这次，调用函数没有了，但是残留了两行代码:

```
new Object[1][0] = "AAA";
Object[] objArr = new Object[]{"CCC", "DDD"};
```

这两行其实就是函数调用的参数，我们的Logger通过可变参数来接收日志，这两句话实际上就是构造了可变参的数组，并进行初始化赋值。

> 我们希望这两句话也没有，怎么处理？

## 彻底擦除

既然方法调用能被擦除，说明配置是没问题的，我只需要不生成这个可变参的构造过程，就可以实现整体擦除。要去除可变参的构造过程是不太可能的；
我们可以通过提供多个同名，但是参数个数不同的方法，来满足常见的可变参调用场景，比如我们提供1-3个参数方法：

```java
public class Logger {
    private final static String TAG = "XXLog";

    public static void info(String msg) {
        if (BuildConfig.DEBUG) {
            Log.i(TAG, msg);
        }
    }

    public static void info(String msg, Object arg1) {
        if (BuildConfig.DEBUG) {
            Log.i(TAG, String.format(msg, arg1));
        }
    }

    public static void info(String msg, Object arg1, Object arg2) {
        if (BuildConfig.DEBUG) {
            Log.i(TAG, String.format(msg, arg1, arg2));
        }
    }

    public static void info(String msg, Object... args) {
        if (BuildConfig.DEBUG) {
            Log.i(TAG, String.format(msg, args));
        }
    }
}
```

再次反编译，可以发现, 整个日志都被擦除了：

```java
public class CustomApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
    }
}
```
## 其他方案

除了利用proguard擦除日志，也可以通过源代码删除的形式，比如通过命令/脚本遍历所有源代码，根据需要移除代码的特征，简历正则表达式，同时剔除命中的代码即可。
