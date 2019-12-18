---
layout: post
title: "Flutter视频滚动播放解决方案"
description: ""
header_image: /assets/img/2019-11-29-01.jpg
keywords: ""
tags: []
---
{% include JB/setup %}
![img](/assets/img/2019-11-29-01.jpg)
* 目录
{:toc #markdown-toc}

如题，本文分享的内容为：视频列表滚动播放。

 视频列表的播放规则一般需要和具体产品、交互确认，播放一般都是静音的，根据露出坐标规律，常见的有两大类：

> 固定位置播放

如滑动屏幕的中间位时，延迟若干毫秒自动播放。

> 固定索引+屏占比播放

如第一个符合屏占比的视频可以自动播放；屏占比可以是当前视频组件的高度百分比，也可以是屏幕上的固定位置；当我们把屏占比定位60%时，第一个视频的可见区小于60%会暂停播放，触发可见的下一个视频。

## 有料视频流
在开发`安居客-有料`内容Feed流时，我们遇到的交互是第二种，由于视频贴出现不固定，待播放的位置也不固定。
在这种情况下要实现与Native一致的效果是一个不小的难题。

我们在开发过程中将这个问题进行了分解：
* 滚动检测：抛开视频，单纯设计一个组件，能够按需检测特定Widget
* 视频播放组件：引入视频播放，将视频控件根据滚动检测要求，进行接入

### 组件原型
下面看一下我们设计的原型组件效果。
![video-play](/assets/images/video-play.gif)

我们将视频贴子用一个色块进行占位。通过监听滑动事件，当停止后，从屏幕顶端开始，遍历当前视窗范围内的控件，如果是视频控件，并且在屏幕上的露出比符合预期，那么将其返回。供开发者后续处理。

核心流程如下图所示：

![](/assets/images/scroll-detect-inner.png)

基于这个思路，我们开发了一个`ScrollDetectListener`组件：

* 使用ScrollDetectListener包裹ListView，ListView为业务相关的视频流帖子。
* 视频帖子使用MetaConsumer包裹；

![](/assets/images/scroll-dectect-listener.png)

```dart
MetaConsumer(
  index: index,
  data: data,
  builder: (BuildContext context, VideoPlayModel model, Widget child) {
    var play = model.playIndex == index;
    return Container(
      width: MediaQuery.of(context).size.width,
      alignment: Alignment.center,
      height: 100,
      padding: EdgeInsets.zero,
      child: play ? Text('Playing $data') : Text('$data'),
      color: play ? Colors.redAccent : Colors.grey[100] ?? Colors.grey,
  );
})
```

### 视频检测组件
通过原型完成必要的调试和优化之后，我们可以尝试接入视频控件。

| Demo1 | Demo2 |
| ----- | ----- |
| ![](/assets/images/video-play-2.gif)| ![](/assets/images/video-play-demo.gif) |

在处理视频的时候，需要注意的点也很多，比如：
* 视频贴的首帧预览图/封面图，采用视频首帧，会遇到视频开头是黑屏的问题，采用独立封面图相对效果会更好。
* 视频高宽比控制
* 帖子状态控制：待播放，加载中，播放，循环/静音等

![](/assets/images/scroll-detect-sample.png)

## 其他方案
除了我们的实现方案，目前还可以找到的一个开源库：[inview_notifier_list](https://github.com/rvamsikrishna/inview_notifier_list)。

这个开源库它支持固定位置播放，也就是我们说的以第一种类型。

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

![](/assets/images/inview-flow.png)

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

## 小结
这两种方案，在检测视频贴的逻辑上分别采用了主动检测和被动检测；
最大差异是我们没有将视频贴提前添加到集合中，所以不用维护一个固定容量的缓存集合。

目前我们的视频滚动播放方案已经完成组件改造，很快将对外开源，敬请期待。

## 参考
* https://github.com/rvamsikrishna/inview_notifier_list/blob/master/README.md


