---
layout: post
title: "Flutter转场动画二三事"
description: "实现转场动画的不同方式"
header_image: /assets/img/2019-08-27-01.jpg
keywords: ""
tags: [Flutter]
---
{% include JB/setup %}
![img](/assets/img/2019-08-27-01.jpg)

* 目录
{:toc #markdown-toc}

## 背景
在开发Flutter应用的时候，如果我们使用了路由配合多页面，必然会涉及到页面转场问题。
在`flutter.dev`有一篇文章介绍了如何实现转场[Animate a page route transition](https://flutter.dev/docs/cookbook/animation/page-route-animation)。

在使用过程中还是有一些不足，比如没有办法和route配置表联合使用，接下来我们先从官方教程来看下，实现转场会遇到哪些问题。

## 自定义转场动画指南
简单回顾下官方教程提供的转场，[Animate a page route transition](https://flutter.dev/docs/cookbook/animation/page-route-animation)。
> 实现从下往上的入场动画

![](/assets/images/page-route-animation.gif)

转场主要使用PageRouteBuilder，通过自定义，提供具体的动画，一共分为以下几个步骤：
* 创建一个PageRouteBuilder实例
* 创建一个补间动画Tween
* 添加一个AnimatedWidget
* 使用CurveTween
* 联合使用两个Tween

*从下往上的动画*

```dart
Route _createRoute() {
  return PageRouteBuilder(
    pageBuilder: (context, animation, secondaryAnimation) => Page2(),
    transitionsBuilder: (context, animation, secondaryAnimation, child) {
      var begin = Offset(0.0, 1.0);
      var end = Offset.zero;
      var curve = Curves.ease;

      var tween = Tween(begin: begin, end: end).chain(CurveTween(curve: curve));

      return SlideTransition(
        position: animation.drive(tween),
        child: child,
      );
    },
  );
}
```

在转场的时候我们，可以作为参数设置给Navigator调用

```dart
RaisedButton(
  child: Text('Go!'),
  onPressed: () {
    Navigator.of(context).push(_createRoute());
  },
)
```

这个实现封装确实能够满足单页面的个性动画，但是他的问题也比较明显：
1. `_createRoute`与页面耦`Page2`合了，不利于动画复用
2. `PageRouteBuilder`需要作为`push`的参数使用，不支持路由表`route`的配合函数`pushNamed`

## 动画复用问题
问题`1`比较好解决，我们可以进一步调整封装，比如把动画提取出来。

下面我们实现一个从右往左的入场动画：

```dart
/// transition: right => left
class SlideRightRoute extends PageRouteBuilder {
  final Widget widget;
  SlideRightRoute({this.widget})
      : super(
    pageBuilder: (BuildContext context, Animation<double> animation,
        Animation<double> secondaryAnimation) {
      return widget;
    },
    transitionsBuilder: (BuildContext context,
        Animation<double> animation,
        Animation<double> secondaryAnimation,
        Widget child) {
      return new SlideTransition(
        position: new Tween<Offset>(
          begin: const Offset(1.0, 0.0),
          end: Offset.zero,
        ).animate(animation),
        child: child,
      );
    },
  );
}
```

使用的时候，只需要把目标页面作为Widget参数传入即可

```dart
RaisedButton(
  child: Text('Go!'),
  onPressed: () {
    Navigator.of(context).push(SlideRightRoute(Page2()));
  },
)
```
## 路由配置表兼容处理
问题`2`的处理，一开始想到的办法比较麻烦，我们需要实现`onGenerateRoute`接口，

这样的好处是，可以做成统一的默认动画，不需要每个页面跳转的时候单独添加一个`PageRouteBuilder`。
```dart
  onGenerateRoute: (RouteSettings setting) {
    var name = setting.name;
    return SlideRightRoute(widget: EmptyScreen('404\n$name'));
  },
```
通过源码分析，我们知道`route`作为默认的路由配置表具有最高优先级，如果通过pushNamed提供的key找不到对应页面，那么会依次命中onGenerateRoute,onUnknownRoute。

> pushNamed('key') => route map => onGenerateRoute => onUnknownRoute

很快遇到了新的问题，通过onGenerateRoute接管后，页面间传参不好使了
```dart
val arg = ModalRoute.of(context).settings.arguments
```
这里原本是可以拿到参数的，现在是空。
要解决这个问题，只有更深入去挖掘route的使用和onUnknownRoute的使用情况。在这个挖掘过程中，意外发现了更简单的控制方法。

## 一键修改默认转场动画
`MaterialApp`有一个属性叫做theme，可以设置默认动画。

```dart
MaterialApp(
  theme: ThemeData(
      pageTransitionsTheme: PageTransitionsTheme(
          builders: <TargetPlatform, PageTransitionsBuilder>{
        TargetPlatform.android: CupertinoPageTransitionsBuilder(),
        TargetPlatform.iOS: CupertinoPageTransitionsBuilder(),
      })),
))
```
通过修改`PageTransitionsTheme`的赋值，我们甚至可以在不同平台的默认转场动画,这里可以自定义`PageTransitionsBuilder`实现。

Flutter提供的默认转场如下：
* Android默认为上下往上的Hexo动效
* iOS默认为左右侧滑动效

```dart
static const Map<TargetPlatform, PageTransitionsBuilder> _defaultBuilders = <TargetPlatform, PageTransitionsBuilder>{
  TargetPlatform.android: FadeUpwardsPageTransitionsBuilder(),
  TargetPlatform.iOS: CupertinoPageTransitionsBuilder(),
};
```

## 小结

现在我们知道了两种修改转场的方法，在开发中可以根据实际需要选用不同方案；
1. 全局默认转场修改，用PageTransitionsBuilder
2. 个别页面特殊动画，用PageRouteBuilder

* [https://flutter.dev/docs/cookbook/animation/page-route-animation](https://flutter.dev/docs/cookbook/animation/page-route-animation)