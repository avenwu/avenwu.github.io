---
layout: post
title: "Flutter吸顶位置优化"
description: ""
header_image: /assets/img/2019-11-24-01.jpg
keywords: "吸顶位置控制"
tags: [flutter]
---
{% include JB/setup %}
![img](/assets/img/2019-11-24-01.jpg)

* 目录
{:toc #markdown-toc}

在Flutter中使用AppBarLayout和SliverPersistentHeader都可以做出基本的吸顶效果。AppBarLayout内部使用的也是SliverPersistentHeader，可以把它理解为一个Framework定制好的状态栏控件，支持较多设置属性，包括吸顶和float等。

当吸顶遇到刘海屏时，事情就会变得有些复杂，一般来说我们希望：
1. 在没有吸顶前，尽可能多的利用屏幕，也就是需要使用沉浸式；
2. 当吸顶的时，吸顶块需要在刘海之下，也就是空出安全区；

## 常见吸顶交互
先看一下结合AppBarLayout，能做的常见的吸顶效果。

![](/assets/images/sliver-pinned-demo1.gif)
![](/assets/images/sliver-pinned-demo2.gif)

这些效果都很好的处理的安全区问题，但如果我们要实现一个Sticky的吸顶效果，但是不要吸顶后的标题栏要如何实现？

## 定义吸顶内容
一个模块吸顶后UI效果可能前后一样，也可能不一样。不一样的情况可以通过引入另一层视图，当吸顶后在展示出来。针对刘海屏的情况，比如iOS的安全区是44，我们吸顶前控件高度可能是40，吸顶后也是40，但是40吸顶的位置需要正好在刘海之下。这就引入了本文要解决的问题：

1. 如何实现指定位置的吸顶？
2. 如何适配不同的指定位置？

我们以实现下面的简单交互进行讨论。指定的高度等于安全区的高度：

![](/assets/images/sliver-pinned-demo3.gif)

## 指定位置吸顶
通过源码分析，发现SliverPersistentHeader并不支持吸顶Y轴的控制。

为了实现这个效果，可以引入一个占位的吸顶块。在模板吸顶之前设置为透明，当目标区不断接近吸顶位置时，我们慢慢将占位块显示出来。

进一步的，为了让渐变的阈值与真实吸顶块滑动速度保持线性关系，我们需要计算真实吸顶块的滑动位置，然后控制占位吸顶块。那么如何计算呢？
SliverPersistentHeader的shrinkOffset参数，只能在目标块开始进入吸顶时才会变化。如果要实现跨块的滑动监听，可以考虑使用ScrollController。然后进行透传。

当然，我们并没有这么处理。在分析视差移动时，发现可以通过引入Stack来使用Position的top属性，控制视图的上边缘溢出高度。

因此我们可以让占位吸顶块的内容一分为二，顶部为真实占位高度，底部为透明的重叠区。这样占位区本身的shrinkOffset偏移量就足够我们计算偏移百分比，只需要让占位吸顶块之后的内容向上偏移相同高度。

![](/assets/images/place-holder-area.png)

通过区域重叠控制，我们使得偏移百分可以扩大计算位置，但是又不影响视觉效果。

最后还有一个问题需要解决，Flutter有一个Overflow的警告机制，如果视图使用不合理，出现了不可见的溢出区域，那么会出现一个黄色警告条。这里，我们认为设置的重叠区就会命中警告。为了结局重叠溢出警告，可以通过Expand控件解决。

![](/assets/images/sliver-pinned-demo4.gif)

## 多端一致适配
Flutter的最大特点就是多段一致，纳闷针对刘海问题，最简单的高度适配，就是动态获取刘海高度，作为指定吸顶的位置。一般可以通过一些属性获得安全区高度（刘海高度）
```dart
MediaQuery.of(context).padding.top
```

## 参考
* [CustomScrollView和Sliver系列](https://github.com/SmallStoneSK/flutter_training_app/tree/master/lib/sliver_widgets)
* [How to Fix the error "BOTTOM OVERFLOWED BY XX.XX PIXELS" in Flutter?](https://fluttercentral.com/Articles/Post/1167)