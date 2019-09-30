---
layout: post
title: "Flutter实现Android Toast组件"
description: ""
header_image: /assets/img/2019-09-20-01.jpg
keywords: ""
tags: [Flutter]
---
{% include JB/setup %}
![img](/assets/img/2019-09-20-01.jpg)

* 目录
{:toc #markdown-toc}

## 背景
Flutter本身没有提供Android的Toast组件，为了使用Toast，可以引入三方库，比如fluttetoast。

但是这个库本身有几个问题：
1. 需要通过MethodChannel来借助原生实现
2. 存在androidx的适配问题

## Flutter实现
为了一劳永逸的规避AndroidX问题，我们移除了对plugin版本的toast依赖，转而使用Flutter来实现全局的Toast效果。

实际上通过OverlayEntry的系统能力，我们也可以实现在app内挂载一个顶级Toast

优势：
1. 轻量级，不需要Android、iOS原生助力
2. 使用简单，不需要配置复杂参数（toast一般都不定制）

我们通过一张图来简单总结下OverlayEntry实现过程中的关键点：

![](/assets/images/toast-with-overlay.png)

1. 展示toast时，通过context获取OverlayState
2. OverlayState可以用于插入一个OverlayEntry，而OverlayEntry是可以承载Widget的容器
3. 移除toast时，通过OverlayEntry的remove操作
4. 为了实现多次toast的不会出现重叠，我们可以服用一个OverlayEntry，当更新时触发markNeedsBuild即可
5. 为了展示效果，可以加一个淡入淡出的动画控制Widget

```dart
class Toast {
  static OverlayEntry _overlayEntry;
  static bool _showing = false;
  static DateTime _startedTime;
  static String _msg;
  static const int INTERVAL = 2000;

  static void toast(BuildContext context, String msg,
      {WidgetBuilder builder}) async {
    assert(builder != null || msg != null);
    OverlayState overlayState = Overlay.of(context);
    _msg = msg;
    _startedTime = DateTime.now();
    _showing = true;
    if (_overlayEntry == null) {
      _overlayEntry = OverlayEntry(
          builder: (BuildContext context) => AnimateLayout(
                builder ?? (BuildContext context) => SimpleToastView(_msg),
                _showing,
              ));
      overlayState.insert(_overlayEntry);
    } else {
      _overlayEntry.markNeedsBuild();
    }
    await Future.delayed(Duration(milliseconds: INTERVAL));

    if (DateTime.now().difference(_startedTime).inMilliseconds >= INTERVAL) {
      _showing = false;
      _overlayEntry.markNeedsBuild();
      Future.delayed(Duration(milliseconds: INTERVAL), () {
        if (!_showing) {
          _overlayEntry.remove();
          _overlayEntry = null;
        }
      });
    }
  }
}
```

使用Toast
```dart
showToast(context, "Hello World");
```
![](/assets/images//Screenshot_1569830999.png)

## 小结
这里实现toast的核心在于系统提供了Overlay的api，同时结合context实现了类似Android的toast

## 参考
* [https://www.jianshu.com/p/cf7877c9bdeb](https://www.jianshu.com/p/cf7877c9bdeb)