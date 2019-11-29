---
layout: post
title: "Fluttet视频列表滚动播放"
description: ""
header_image: /assets/img/2019-11-29-01.jpg
keywords: ""
tags: []
---
{% include JB/setup %}
![img](/assets/img/2019-11-29-01.jpg)
* 目录
{:toc #markdown-toc}

如题，本文分享的主题为：视频列表滚动播放。

这种类似效果在原生开发中比较常见了，主要交互如下：
* 视频流/混合流中视频露出后，静默播放（一般是静音的）
* 一屏出现多个视频时，需要定义播放规则

## 讲讲播放规则
播放规则一般需要和具体产品、交互确认，播放一般都是静音的，根据露出坐标规律，常见的有两大类：

> 固定位置播放

比如滑动屏幕的中间位时，延迟若干毫秒自动播放。

> 固定索引+屏占比播放

比如第一个符合屏占比的视频可以自动播放；屏占比可以是当前视频组件的高度百分比，也可以是屏幕上的固定位置；比如说第一个视频的可见区小于60%时，暂停播放，触发可见的下一个视频。

## 规则与算法
不同的播放规则，采用的实现方案可能就不一样。目前可以找到的一个开源解决方案是：[inview_notifier_list](https://github.com/rvamsikrishna/inview_notifier_list)。

它支持固定位置播放，也就是我们说的以第一种类型。

![](/assets/images/59606620-3c241980-912f-11e9-8c63-3029661c76ac.png)

该项目提供了预览效果和demo，非常良心的作者。

| Demo1 | Demo2 |
| ----- | ----- |
| ![](/assets/images/59602739-2f022d00-9125-11e9-84ef-19a33f8bd782.gif)| ![](/assets/images/59602740-2f022d00-9125-11e9-8ee6-044e44f6048f.gif) |

他的基本原理如下：
* 设计了一个InViewState容器，继承了ChangeNotify，用于记录列表中的子Widget的context信息，`InViewState state = InViewNotifierList.of(context);`
* 当视频Widget执行build时，调用InViewState.addContext加入集合中
* 同时视频Widget本身需要用AnimatedBuilder将InViewState和视频状态进行关联，也就是当InViewState更新时，重新构建视频Widget
* 什么时候更新InViewState呢？答案是滑动过程中。滑动的时候开始遍历前面加入的视频Widget的contxt，其实就是为了通过context难道控件最终的坐标，判断是不是符合规则
* 如果坐标位置命中规则，就会吧当前的控件的标识id缓存到一个集合中`_currentInViewIds`，然后通知观察者数据变化了，所以AnimatedBuilder会触发child的build
* 在child的build时（也就是前面提到的视频空间build），就会在判断当前视频的id是否在`_currentInViewIds`中，进行差异化绘制。

我们用一张图来概括性大致流程：

![](/assets/images/inview-flow.png）

可以看到这里面有两个集合，分别用那个有缓存控件的Context和名中控件数据。为了避免遍历的性能问题，作者引入了缓存阈值，可以由使用者根据实际情况调整缓存context的个数。比如我们可以把个数设置为一个屏幕最多能展示的视频条数。这样很可能不超过5个。

```dart
///Add the widget's context and an unique string id that needs to be notified.
void addContext({@required BuildContext context, @required String id}) {
_contexts.add(_WidgetData(context: context, id: id));
}

///Keeps the number of widget's contexts the InViewNotifierList should stored/cached for
///the calculations thats needed to be done to check if the widgets are inView or not.
///Defaults to 10 and should be greater than 1. This is done to reduce the number of calculations being performed.
void removeContexts(int letRemain) {
if (_contexts.length > letRemain) {
  _contexts = _contexts.skip(_contexts.length - letRemain).toSet();
}
}
```

这个实现方案基本满足的场景一的规则。并且实现了局部刷新的监听。

下面我们看下场景二，如果要基于当前视频的位置做计算，这个库支持不了。

可以看到它暴露的参数: 视频控件上边缘差值，下边缘差值值，视窗高度H
```
//Check if the item is in the viewport by evaluating the provided widget's isInViewPortCondition condition.
  isInViewport = _isInViewCondition(deltaTop, deltaBottom, vpHeight);
```

如果我们要计算视频控件自身的露出情况，需要单独获取控件的坐标和大小来计算。如果进行二次开发，只需要将判定函数扩展一些参数即可。

## 其他实现思路

在实际开发中，除了上面的思路，其实还可以通过动态计算找到符合预期的视频控件。我们通过监听滑动事件，当停止后，从屏幕顶端开始，遍历当前视窗范围内的控件，如果是视频控件，并且在屏幕上的露出比符合预期，那么将其返回。供开发者后续处理。

这个思路和上面的区别在于对视频组件的感知是被动的，不需要提前添加到集合中，也不用维护一个固定容量的缓存集合。

![](/assets/images/scroll-video-play.png)

![](/assets/images/scroll-dectect-listener.png)

## 参考

* https://github.com/rvamsikrishna/inview_notifier_list/blob/master/README.md


