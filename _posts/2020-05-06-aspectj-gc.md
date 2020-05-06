---
layout: post
title: "AspectJ切面实例模式与GC的探究"
description: "AspectJ切面实例与GC垃圾回收问题"
header_image: /assets/img/2020-05-06-01.jpeg
keywords: ""
tags: [AspectJ]
---
{% include JB/setup %}
![img](/assets/img/2020-05-06-01.jpeg)

* 目录
{:toc #markdown-toc}

## 切面代码初探

通过AspectJ在编译期间生成代码，并根据我们的JointCut和Advice在目标位置进行代码的织入/插桩。所以第一个感兴趣的点就是插桩的代码生成的效果是怎么样的。

切入点的选择是可枚举的：**Before, After, AfterReturning, AfterThrowing, and Around**

常用的是在某一个方法执行前/后插入代码，用于插桩统计代码调用情况或者日志打点。根据调用时机又可以分为call还是execution。针对匹配的插入点还可以结合通配符，与或非等逻辑运算进行过滤。整个切面编程的细节其实很多，完整掌握可以去经常查阅AspectJ的开发手册[The AspectJTM 5 Development Kit Developer's Notebook](https://www.eclipse.org/aspectj/doc/next/adk15notebook/index.html)

本文不谈如何使用AspectJ，读者可以自行百度相关上手资料。我们主要通过分析来解答一下几个问题：

* AspectJ切面的实现代码与GC
* AspectJ的切面实现

## 切面实例与使用

默认生成的切面逻辑封装在一个单例对象中。有多个Aspect切面则会有多个单例。

**@Aspect**注解可以接受一个value参数，默认值为空“”代表单例模式。相关API说明可见java doc

```java
/**
 * Aspect declaration
 *
 * @author Alexandre Vasseur
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Aspect {

    /**
     * @return the per clause expression, defaults to singleton aspect.
     * Valid values are "" (singleton), "perthis(...)", etc
     */
    public String value() default "";
}
```

可知通过赋值为`perthis(...)`即可实现非单例的实例效果。在一个Classloader中，单例从生到死都是不会被GC的，如果独立切面很多，则会出现很多单例，因此需要避免切面持有上下文或者大对象。

```
@Aspect
public class Foo {}
```
```
@Aspect("perthis(execution(* abc..*(..)))")
public class Foo {}
```

不使用注解，通过切面代码的等价写法如下：
```
public aspect Foo {}
```
```
public aspect Foo perthis(execution(* abc..*(..))) {}
```

## 切实代码

根据切面是否为单例实例模式，其生成的代码是不同的。下面看一下两种情况下生成的代码。

为了使得分析有些实际意义，首先定义插桩场景：

> 统计线程被start的调用情况，在有新的线程start时，打印出上下文

随意写了一个线程实例化+启动的逻辑如下所示：

```java
private Thread newThread() {
  return new Thread();
}
// 在Thread的start调用前插桩
newThread().start();
```

```java
@Aspect
public class ThreadAspectJ {

    @Pointcut("!within(cn.hacktons.core.DelayAspectJ)")
    public void ignore() {
    }

    @Pointcut("call(Thread+.new(..)) && ignore()")
    public void constructor() {
    }

    @Pointcut("call(void Thread+.start()) && ignore()")
    public void threadStart() {
    }

    @Pointcut("call(java.util.concurrent.ThreadPoolExecutor+.new(..)) && ignore()")
    public void executor() {
    }

    @Pointcut("call(static * java.util.concurrent.Executors.*(..))")
    public void executorsMethod() {
    }

    @Before("constructor() || threadStart() || executor() || executorsMethod()")
    public void beforeThreadNew(JoinPoint point) {
        Helper.enterMethod(point);
    }
}
```

### singleton实例模式

经过ajc编译后，插桩的代码如下：

```java
Thread newThread = newThread();
ThreadAspectJ.aspectOf().beforeThreadNew(Factory.makeJP(ajc$tjp_2, this, newThread));
newThread.start();
```

单例实例模式是默认值，因此无线额外配置，器插入的代码是在切入点调用aspectOf获取单例对象并执行我们编写的advice函数。

这个单例的初始化是在切面类的静态块中。

![](/assets/images/aspectj-singleton.png)

由于是单例，并且没有任何置空的逻辑，因此该切面一旦被引用，将在App的整个后续生命周期中支持存在并占用内存空间。可以通过Profiler和MAT观察GC后的内存对象加以验证，在本案例中，切面没有显式持有任何业务上的数据他的大小如下：

![](/assets/images/aspectj-instance2.png)

* shallow size 8bytes
* retained size 304bytes

其中shallow代表对象自身大小，retained还包括引用对象大小，可以认为是GC后能释放的空间。

![](/assets/images/aspectj-instance3.png)

如果存在多个切面类，对应的就存在多个单例。

> PS: 这里的size和MAT有些差异, MAT中不同面板显示的shallow size也不尽相同，偏小的值从数据上看应该是取自objectSize

![](/assets/images/aspectj-instance.png)

### perthis实例模式

要修改默认的单例切面，需要为Aspect注解添加perthis(...)其中...指代切如点匹配规则。在本文案例中，可简单调整为如下

```
@Aspect("perthis(constructor() || threadStart() || executor() || executorsMethod())")
```


```java
// 插桩代码逻辑
Thread newThread = newThread();
ThreadAspectJ.ajc$perObjectBind(this);
if (ThreadAspectJ.hasAspect(this)) {
  ThreadAspectJ.aspectOf(this).beforeThreadNew(Factory.makeJP(ajc$tjp_2, this, newThread));
}
newThread.start();

// 被插桩的类MainActivity
public class MainActivity extends AppCompatActivity implements C0844ajcMightHaveAspect {
  private transient /* synthetic */ ThreadAspectJ ajc$cn_hacktons_core_ThreadAspectJ$perObjectField;

  public /* synthetic */ ThreadAspectJ ajc$cn_hacktons_core_ThreadAspectJ$perObjectGet() {
    return this.ajc$cn_hacktons_core_ThreadAspectJ$perObjectField;
  }

  public /* synthetic */ void ajc$cn_hacktons_core_ThreadAspectJ$perObjectSet(ThreadAspectJ threadAspectJ) {
    this.ajc$cn_hacktons_core_ThreadAspectJ$perObjectField = threadAspectJ;
  }
}
```

插桩的代码含义比较直白：

1. 调用了静态方法ajc$perObjectBind，并将当前对象的引用传入绑定
2. 判断是否存在切面
3. 获取切面实例对象，并执行advice函数beforeThreadNew

在这三步中，只有beforeThreadNew函数是我们显式编写的，其他都是ajc自动生成的模板代码。

可以看到被插桩的类被添加了一个继承关系，继承的接口为切面类中自动生成的C0844ajcMightHaveAspect。

我们看一下绑定函数的实现：

```java
public static synchronized /* synthetic */ void ajc$perObjectBind(Object obj) {
    synchronized (ThreadAspectJ.class) {
        if ((obj instanceof C0844ajcMightHaveAspect) && ((C0844ajcMightHaveAspect) obj).ajc$cn_hacktons_core_ThreadAspectJ$perObjectGet() == null) {
            ((C0844ajcMightHaveAspect) obj).ajc$cn_hacktons_core_ThreadAspectJ$perObjectSet(new ThreadAspectJ());
        }
    }
}
```
在绑定函数中根据当前传入的对象是否已经绑定了切面实例，会进行一次切面初始化并执行setter。到这里我们基本了解了模板代码的思路，getter的使用也就比较简单了，通过aspectOf方法，传入被插桩的类实例，来获取其绑定的切面类。

我们把这块逻辑稍微整理下，可以得到如下的关系图：

![](/assets/images/aspectj.png)


因此针对`perthis`模式来说，**一个插桩目标类中如果存在多个切面，那么这些切面共享同一个切面实例。**


## 小结

基于对插桩生成代码的分析，可以得到的结论如下：

* 切面默认已单例实现，所以同一个Classloader下共享同一个实例对象，且不会被GC回收
* 如果切面类较多，并且使用时机比较单调，可以考虑使用perthis模式，随着被插桩对象释放可以被GC回收
* 不同实例模式下生成的模板代码略有差异，单例模式下模板代码更少；perthis模式下会生成协议接口，并自动集成&实现
* 除了上述两种模式在Aspect手册中还有pertypewithin模式，从文档来看与perthis差异主要在配置规则上？

## 参考

* [Chapter 9. An Annotation Based Development Style - **Aspect Declarations**](https://www.eclipse.org/aspectj/doc/next/adk15notebook/ataspectj-aspects.html)
* [Chapter 9. An Annotation Based Development Style - **aspectOf() and hasAspect() methods**](https://www.eclipse.org/aspectj/doc/next/adk15notebook/ataspectj-aspectof.html)
* [Garbage collection of aspect instances](https://www.eclipse.org/lists/aspectj-users/msg06830.html)
