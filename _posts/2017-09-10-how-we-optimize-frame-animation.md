---
layout: post
title: "帧动画调优实践"
description: "解决Android原生帧动画AnimationDrawable的内存溢出OutOfMemmory问题"
header_image: /assets/wuba/Android%E5%8E%9F%E7%94%9F%E5%B8%A7%E5%8A%A8%E7%94%BB.png
keywords: "帧动画，内存溢出"
tags: [android]
---
{% include JB/setup %}
![img](/assets/img/2017-09-11-01.png)

本文源于笔者内部分享整理而来，同步发表于：

* https://www.zybuluo.com/avenwu/note/876161
* 微信公众号：App架构

---

帧动画调优实践
=======

## 1. 背景

在做动画的时候我们有很多选择方案。

* 最常见的是Android原生的帧动画，位移动画，旋转动画，属性动画等等，具体根据动画效果选择实现方案；
* 针对那些有规律，不太复杂的矢量动画我们往往也采取自定义View来实现，比如各种狂拽炫酷的loading动画；
* 如果自定义实现成本比较大，或者难以达到Android/iOS多端统一，也经常采用帧动画，由UE提供帧动画素材；
* 如果项目支持，也可以使用Airbnb出品的Lottie[^Lottie]，这是一个非常牛x的动画解析库，不过对sdk版本有要求，具体可以自行查验下.

目前最优方案是使用Lottie动画库，在无法使用Lottie的情况下，我们就需要手动来具体帧动画的调优了。

本文主要谈谈做帧动画的一些优化策略以避免`OutOfMemoryError`问题。

## 2. 帧动画有什么问题？

我们都知道原生的Android帧动画在加载序列帧时，是一次性将所有序列帧的图片编码到内存当中的，所以执行帧数较多的动画时很容易发生内存不足，抛出 `OutOfMemoryError`。

所以呢，你就需要解决内存问题，基本上可以从以下几方面入手；

* 降帧
> 在保证UI效果和视觉流畅度的情况下尽可能减少帧数，比如UE输出的序列帧可能默认有有4，5十张图片，减帧后可能只有20张；这里一定要注意的是，必须经过UE来减帧，RD不允许私自调整，避免效果不达标；

* 压缩尺寸
> 根据在手机上的展示大小，缩放到一致的高宽，避免屏幕上展示100x100,你用一个500x500的资源；

* 压缩体积
> 采用无损或者可接受的有损压缩图片质量，这个也无需多说，老司机都懂得。打个广告，推荐笔者写的IntelliJ插件 [http://avenwu.net/biu/](http://avenwu.net/biu/);

* 重写帧动画的编码逻辑
> 这里的一个思路是动态编码图片，不再一次性加载所有图片，通过懒加载的方式，图片对内存的要求会大幅降低；

## 3. 手工实现帧动画
### 3.1 帧动画分析

下面主要针对 `重写帧动画的编码逻辑` 这个维度来聊聊具体的实现策略。
既然要重写帧动画，首先就要知道Android原生帧动画的实现逻辑；通过阅读相关源码，笔者绘制了如下简化示意图，基本涵盖了帧动画的构造和执行过程；

![Android原生帧动画](/assets/wuba/Android%E5%8E%9F%E7%94%9F%E5%B8%A7%E5%8A%A8%E7%94%BB.png)

可以看到帧动画在首次inflate的时候会解析xml，并将每一个item节点解析为drawable对象实例，然后加入到数组当中；后续动画过程就是轮询绘制，在Drawable容器中绘制当前drawable即可，整体代码简洁漂亮。

但是这个逻辑我们并不能直接复用到手工实现的帧动画中，为什么？这里暂不解答，读者可以自己思考一下：）

现在我们把帧动画的实现策略抽象一下，它就是一个生产者和消费者的关系，如下图所示：

![帧动画模型](/assets/wuba/%E5%B8%A7%E5%8A%A8%E7%94%BB%E6%A8%A1%E5%9E%8B.png)

总的来说我们的优化策略集中在编码和缓存上面，渲染不需要改变。根据编码的的策略可能代码差异会很大。在实际开发中，我们也遇到过一些坑，比如编码速度跟不上，导致渲染的时候出现跳帧，推测主要是CPU时间切片问题。
所以如果做成单线程编码，UI线程渲染是要非常谨慎的，否则很有可能你的编码业务全部被阻塞了，导致UI层面频繁丢帧。

### 3.2 Bitmap 复用

在图片编码之后，我们需要考虑Bitmap的编码复用，这样可以避免每次decode一张图片都要全新申请一块内存，通过复用已有的bitmap实例对象我们可以做到编码的更低开销。

其核心在于 `Options#inBitmap`，这个API是Android引入的一个新API，新API本身没什么问题，问题在于这个属性是API 11引入的，但并不可以直接使用，这导致做版本判断的时候比较坑（你无法确定这个接口是否可用,除非你穷举一下相关SDK版本）。根据源代码，在API 14的时候使用该属性会抛异常，具体大家可以去develop了解相关细节，。

![bitmap复用](/assets/wuba/bitmap%E5%A4%8D%E7%94%A8.png)

在引入了Options#inBitmap的时候一定要注意姿势。否则的话极有可能出现大量日志警告：

```
Called reconfigure on a bitmap that is in use! This may cause graphical corruption!
```

当然如果出现了这个警告也不要紧张，肯定是可以解决的，这个警告是在`core/jni/android/graphics/Bitmap.cpp`中抛出来的，含义其大致就是说如果你在修改一个正在被使用的bitmap那么这会导致`graphical corruption`。这个单词没有想到合适的中文翻译，我猜就是会破坏图形的意思吧。

如果你一定要忽略这个异常的话，请忍受控制台的大量警告输出，反正我是忍不了。要解决整个警告也很简单，只要保证复用bitmap作为inBitmap时不要使用当前正在被ImageView展示的bitmap实例就可以了。

### 3.3 RAM和CPU的平衡处理

前面提到了，编码图片可能会跟不上渲染，这是因为编码一张图片需要CPU去处理图片，包括IO之类的，这需要编码线程申请CPU时间切片，如果UI线程或者其他线程池有高优的任务，那么编码这一块就会很尴尬。所以要解决这个问题一方面可以提供bitmap内存缓存，减少需要重新编码的次数，比如我们通过调试把图片复用的命中率提高到40%,50%之类。另外一个就是不能给编码线程设置`过低的优先级`，不要觉得这是后台任务应该放一个BACKGROUND之类的低优先级，根据andorid的官方说明（具体文档我忘了），后台线程和前台（其他默认优先级线程）之间获取CPU的能力差异是非常大的，好像接近二八开的样子。

### 3.4 缓存策略

缓存策略也是需要认真考虑的地方，前面我们讲到了为了平衡编码压力，我们可以引入内存缓存，那么以什么策略来缓存图片呢？（在心里说出一个答案，看看对不对）
> 
**LRU？**

如果你回答LRU的话，恭喜你，你的缓存完全失效了。

虽然LRU使用非常广泛的缓存算法，但是很遗憾他并不适合帧动画缓存，为什么呢？读者可以自己思考一下LRU的淘汰策略和帧动画的执行策略就明白了。

所以我们用的是什么装逼的算法呢？这个笔者并不是专研算法的，所以不懂得太多唬人的名字。不过回想起多年前求学时，耳(quan)熟（bu）能（wang）详（ji）的那些算法，你会想起还有一个叫LIFO的东西，也就是后进先出算法（Last in first out）。

![缓存策略](/assets/wuba/%E7%BC%93%E5%AD%98%E7%AD%96%E7%95%A5.png)

我们根据需要可以稍加改造找一下，首先实现一个LifoCache,这个可以从LruCache略加改造，然后在取缓存/删缓存时，取倒数第二个即可。这样可以最早执行的动画被缓存下来，并且有机会再下一轮动画来临时被复用。当然机智的你也可以选择其他更好的算法。

### 3.5 优化点回顾

以上是我们在实践当中总结的一些比较关键的点。出于尊重原创的职业素养，必须声明，我们的整体优化的启示灵感来源于`FasterAnimationsContainer[^FasterAnimationsContainer]`；
原项目更多的像一个示意demo，存在诸多bug，需要修复才能满足基本使用，因此我们修复了这些问题，并且发起了 `Pull Request[^PR]`。

不过从维护记录来看原项目已经不太活跃，我们将这些改动和新增的功能优化做了梳理。
当然除了修复这些问题，我们实际上进行了几乎95%以上的重构，所以严格来说，这两个项目除去解决内存泄漏的思想，在工程上已经没有太多相似处了。

我们主要做了如下改动：

> 
1. 消除bitmap编码警告；
2. 修复前后台切换后动画僵死的问题；
3. 取消单例模式，支持每个ImageView控制独立的动画；
4. 新增图片内存缓存，支持缓存比配置；
5. 动画不显示时缓存自动释放与恢复；
6. 支持原生xml定义，兼容原生写法；


## 4. 调优效果

这一节，我们简单对比下，手工实现的帧动画和原生帧动画的内存数据；

### 4.1 原生标准帧动画

![原生标准帧动画](/assets/wuba/%E6%A0%87%E5%87%86%E5%B8%A7%E5%8A%A8%E7%94%BB.png)

在这幅图中，我们执行了一个完整的操作流程

- 启动程序（不包含目标动画视图）；
- 跳转到原生帧动画Activity，内部有一个ImageView，设置了android:src为一个xml帧动画；
- 可以看到刚进入目标Activity内存立刻开始大幅上升，CPU出现了波动；而此时我们实际上没有开始执行动画；
- 获取ImageView的src并start动画，此时内存没有明显波动，并且CPU也相对平稳；
- 持续播放动画，内存和CPU基本不再变化，维持在一个高位状态；
- 退出页面，GC后内存全部释放；

### 4.2 懒加载帧动画

前面已经提到，我们的帧动画支持缓存设置，所以分别看一下缓存40%和100%缓存的两种情况。

![懒加载帧动画](/assets/wuba/%E6%89%8B%E5%8A%A8%E5%B8%A7%E5%8A%A8%E7%94%BB.png)

我们保持一致的操作流程，来看一下优化后的帧动画的数据；

- 启动程序（不包含目标动画视图）
- 跳转到优化帧动画Activity，内部有一个ImageView，设置了app:src为一个xml帧动画；
- 刚进入目标Activity内存小幅增加，CPU出现波动；此时我们实际上也没有开始执行动画，这个内存开销是预览的首帧；
- 获取ImageView的src并start动画，随着动画的执行内存使用量开始慢慢上爬，期间伴随着CPU的波动；
- 持续播放动画，内存达到一个稳定态，基本不再变化；CPU持续变化；
- 退出页面，GC后内存全部释放；

类似的当我们把缓存比调大，比如调到100%后，可以得到一个新的图。这个图的内存高峰和CPU状态基本可以看做是上面两种情况的结合体。当缓存占比高了，那么后续需要重新编码的次数也就少了，所以CPU的占用也就少了。

![懒加载帧动画-全缓存](/assets/wuba/%E6%89%8B%E5%8A%A8%E5%B8%A7%E5%8A%A8%E7%94%BB-%E5%85%A8%E7%BC%93%E5%AD%98.png)

### 4.3 数据对比

以我们的测试为例，选用了 `16` 张 `600x600` 的jpg图片，每张大小约为 `14~18KB` 之间。
为了方便，对比我们选取几个关键点的近似内存和CPU数据。

| 动画类型     |   阶段   |内存/MB| CPU/% |
| ------------ | :-----:  | :----:|:----:|
| 原生帧动画   | 启动页面 |   65  | 1% |
| 原生帧动画   | 加载动画 |   65  | 1% |
| 懒加载帧动画 | 启动页面 |   17  | 1% |
| 懒加载帧动画 | 加载动画 |   32  | 15%|

## 5. 使用说明

介绍完了原理，就来看下怎么使用吧，API层面没有做太多改变和原生使用比较接近。由于项目是开源的，这里放上传送门：

> [http://hub.hacktons.cn/animation/](http://hub.hacktons.cn/animation/)

通过上述地址可以获取项目最新的变化和源代码等信息。 下面我们简单看下目前如何使用我们的优化方案来播放一个帧动画。

### 5.1 配置Gradle依赖库

```
compile 'com.github.avenwu:animation:0.2.0'
```
### 5.2 通过MockFrameImageView使用动画

先进行动画定义，一般都是用一个xml搞定，方便省事。

```xml
<?xml version="1.0" encoding="utf-8"?>
<animation-list
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:oneshot="false">
    <item
        android:drawable="@drawable/num_0"
        android:duration="120"/>
    <item
        android:drawable="@drawable/num_1"
        android:duration="120"/>
    <item
        android:drawable="@drawable/num_2"
        android:duration="120"/>
    <item
        android:drawable="@drawable/num_3"
        android:duration="120"/>
    <item
        android:drawable="@drawable/num_4"
        android:duration="120"/>
    <item
        android:drawable="@drawable/num_5"
        android:duration="120"/>
    <item
        android:drawable="@drawable/num_6"
        android:duration="120"/>
    <item
        android:drawable="@drawable/num_7"
        android:duration="120"/>
    <item
        android:drawable="@drawable/num_8"
        android:duration="120"/>
    <item
        android:drawable="@drawable/num_9"
        android:duration="120"/>
    <item
        android:drawable="@drawable/num_a"
        android:duration="120"/>
    <item
        android:drawable="@drawable/num_b"
        android:duration="120"/>
    <item
        android:drawable="@drawable/num_c"
        android:duration="120"/>
    <item
        android:drawable="@drawable/num_d"
        android:duration="120"/>
    <item
        android:drawable="@drawable/num_e"
        android:duration="120"/>
    <item
        android:drawable="@drawable/num_f"
        android:duration="120"/>
</animation-list>
```

在layout中进行布局定义

```xml
<cn.hacktons.animation.MockFrameImageView
    android:id="@+id/imageview"
    android:layout_width="200dp"
    android:layout_height="200dp"
    android:layout_gravity="center"
    app:cache_percent="0.4"
    app:src="@drawable/loading_animation"/>
```

最后在Activity或者其他地方就可以启动动画了

```java
ImageView imageView = (ImageView) findViewById(R.id.imageview);
animateDrawable = (Animatable) imageView.getDrawable();
((Switch) findViewById(R.id.button)).setOnCheckedChangeListener(new CompoundButton.OnCheckedChangeListener() {
    @Override
    public void onCheckedChanged(CompoundButton buttonView, boolean isChecked) {
        if (isChecked) {
            animateDrawable.start();
        } else {
            animateDrawable.stop();
        }
    }
});
```

上面的使用方法维持了和原生帧动画一致的操作和配置，除了layout里面写的控件不是ImageView，其他一模一样的。

### 5.3 通过代码使用动画

除此之外，我们也可以直接用代码来实现这个动画配置，这也是项目最开始所支持的方式，这个就不需要依赖任何自定义的View，直接在原生ImageView上生效：

```java
int[] FRAMES = {
    R.drawable.num_0,
    R.drawable.num_1,
    R.drawable.num_2,
    R.drawable.num_3,
    R.drawable.num_4,
    R.drawable.num_5,
    R.drawable.num_6,
    R.drawable.num_7,
    R.drawable.num_8,
    R.drawable.num_9,
    R.drawable.num_a,
    R.drawable.num_b,
    R.drawable.num_c,
    R.drawable.num_d,
    R.drawable.num_e,
    R.drawable.num_f,
};
animateDrawable = new AnimationBuilder()
    .frames(FRAMES, 120/*duration*/)
    .cachePercent(0.4f)
    .oneShot(false)
    .into(findViewById(R.id.imageview));

((Switch) findViewById(R.id.button)).setOnCheckedChangeListener(new CompoundButton.OnCheckedChangeListener() {
    @Override
    public void onCheckedChanged(CompoundButton buttonView, boolean isChecked) {
        if (isChecked) {
            animateDrawable.start();
        } else {
            animateDrawable.stop();
        }
    }
});
```

### 5.4 效果对比图

最后看一下各种帧动画的实现效果。

![Animation](/assets/wuba/device-2017-09-12-153056.gif)

[Download Video](/assets/wuba/device-2017-09-12-153056.mp4)

## 6. 小结

综合来看，原生帧动画更适合体量较小，内存压力不那么大的帧动画，如此一次性加载所有帧，可以保证后续帧切换的流程性；而MockFrameAnimation则为了解决内存问题，采取动态编码序列帧。

这相当于用CPU的编码/计算能力换取了内存消耗；同时为了达到适合的平衡，我们允许开发者设置图片缓存的张数，缓存数越大那么内存消耗越多，需要重新编码的次数也就相对更少；

[^Lottie]: https://github.com/airbnb/lottie-android

[^biu]: http://avenwu.net/biu/

[^FasterAnimationsContainer]: https://github.com/tigerjj/FasterAnimationsContainer

[^PR]: https://github.com/tigerjj/FasterAnimationsContainer/issues/11
