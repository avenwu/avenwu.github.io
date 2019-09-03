---
layout: post
title: "Flutter凸起导航栏优化"
description: "实现凸起导航栏的设计"
header_image: /assets/img/2019-09-03-01.jpg
keywords: ""
tags: [Flutter]
---

{% include JB/setup %}
![img](/assets/img/2019-09-03-01.jpg)

* 目录
{:toc #markdown-toc}

## 背景
在导航栏的常见交互设计中，有一种是底部凸起一个按钮，比如居中凸起，也有动态随着选中态凸起的。本文分享在实现凸起导航栏的几种迭代优化思路。

![](/assets/images/screentshot-1565614892170.png)

## 自定义NavigationBar
要实现这种效果，在Flutter内不能通过标准控件实现，常见的`BottomNavigationBar`支持的是方正对齐的TAB。如果你的其中一个要凸起，默认是实现不了的。

![screentshot-1565614784395.png](/assets/images/screentshot-1565614784395.png)

自定义一个Widget用于NavigationBar，接口尽量保持可配，方便复用。如下我们提供了设置tab数组的能力，以及对tab选中态文字的控制，凸起效果利用一个大的背景图实现。

```dart
@override
Widget build(BuildContext context) {
  return Scaffold(
    body: Center(
      child: _widgetOptions.elementAt(_selectedIndex),
    ),
    bottomNavigationBar: HomeBottomNavigationBar(
      items: _navigationViews,
      onTap: _onItemTapped,
      backgroundAsset: 'assets/images/home/wb_tab_bg.png',
      defaultColor: Colors.black,
      selectColor: Colors.red,
    ),
  );
}
```

为了支持TAB的凸起，同时保证外部使用的时候可以和Scaffold搭配在一起，我们需要保证NavigationBar的实现支持层叠效果，即不会导致外部`Scaffold#body`出现底部镂空效果。

为了实现这个效果，可以利用`Stack`控件。我们将导航栏分解为几个部分：
1. 大背景图，中间凸起
2. 普通方正的tab图文控件
3. 放大的tab图文控件

![](/assets/images/navigationbar_split.png)


```dart
return Stack(
  overflow: Overflow.visible,
  alignment: Alignment.bottomCenter,
  children: <Widget>[
    // 作为锚点用的，必须存在一个固定高度的元素，才能配合Position达到clip效果
    Padding(
      padding: EdgeInsets.only(top: 50),
    ),
    Positioned.fill(
      top: -18,
      child: Image.asset(
        widget.backgroundAsset,
        fit: BoxFit.contain,
        height: 61,
      ),
    ),
    Positioned.fill(
      top: -11,
      child: Container(
          alignment: Alignment.bottomCenter,
          height: 61,
          child: Row(
            mainAxisAlignment: MainAxisAlignment.spaceAround,
            crossAxisAlignment: CrossAxisAlignment.stretch,
            children: children,
          )),
    ),
  ],
);
```

## NavigationBar优化
其实到这里tab已经达到了目标效果，但是这个效果依赖图背景图，适配的时候需要注意，比如横竖屏的时候，中间凸起的拉伸问题。除此之外，在我们的场景中有另外一个问题，二级页面相同图标的对齐。
我们用一个小动画看一下

![](/assets/images/Record_2019-09-03-15-16-09.gif)

要是上图效果更好，我们需要解决两个问题：
1. 横竖屏的适配
2. 图标位置精准对齐


### 横竖屏的适配

这个其实很多App是不支持横竖屏的，因此粗暴的办法是禁止横竖屏切换，默认支持横屏或者竖屏。当然我们这里不会采用这个方案。

另外的方案：
1. 使用多张图，当出现横屏时，使用一张特殊的图片
2. 利用图片指定位置拉伸，类似android的nine-patch图片（.9.png）
3. 摒弃背景图方案，通过代码绘制凸起和阴影效果等

### 图标位置精准对齐

这个的实现思路是在第二个页面，通过精确计算，在相同位置摆放一个相同图片。思路就是这么简单，具体做的时候其实也会有一些问题，由于两个页面布局不同，实现方案可能会出现一些奇葩的bug，在下一节我们会具体分析这个bug。

直观来说前后两个页面的布局方案相同或者相似的手，对齐相对容易。最后我们也是基于这个思路对页面二级做了调整来满足图标问题。

## NavigationBar重构

讲解重构思路前，先看一下最终效果，顺便可以和前面的抖动做一下前后对比。

![](/assets/images/Record_2019-09-02-19-15-13.gif)

通过分析源码，我们得知bottomNavigationbar配套可以和floatAcitonButton使用，并且可以同location调整两者的结合形式。

![](/assets/images/flutter-navigation-bar-location.png)

利用系统api，我们可以轻松实现这四种效果，其中`图4`和我们的目标效果有点接近。
```dart
Scaffold(
	floatingActionButton: // button,
	floatingActionButtonLocation:// location,
	bottomNavigationBar: BottomAppBar(
        notchMargin: 0,
        shape: CircularNotchedRectangle(),
        child: new Row(
          mainAxisSize: MainAxisSize.min,
          mainAxisAlignment: MainAxisAlignment.spaceAround,
          children: children,
        )
    )
)
```

基于`图4`要实现我们的目标效果，还需要做几件事情
* 翻转凹陷区域=》实现凸起
* 控制凸起幅度=》调低凸起
* 控制凸起图标=》调整大小

针对这三个点，我们逐一研究。

### 翻转凹陷区域=》实现凸起

原生的凹陷效果，用一个术语叫：`notch`,他的实现类是CircularNotchedRectangle，因此，我们要做的就是自定义一个NotchedShape类，实现对凸起的绘制。

![](/assets/images/navigation-tab-notch.png)

![](/assets/images/navigation-tab-convex.png)

### 控制凸起幅度=》调低凸起

这个核心还是对曲线的控制，可以加入偏移量来实现，这样的好处是不用破坏曲线计算。在处理的时候，注意要考虑对Snackbar和bottomsheet的兼容处理，否则你的按钮回被snackbar顶起来。

### 控制凸起图标=》调整大小

这个字需要对图标大小进行控制即可。

做完这些优化后，看下最终的静态图效果，即使是横屏也一样适配顺滑，切换动画可以看前面的gif。

![](/assets/images/screentshot-1567423132789.png)

## 小结
关于自定义noth的部分，涉及到数学计算，后续我们单独介绍。

## 参考
* [https://proandroiddev.com/flutter-how-to-using-bottomappbar-75d53426f5af](https://proandroiddev.com/flutter-how-to-using-bottomappbar-75d53426f5af)
* [https://medium.com/coding-with-flutter/flutter-bottomappbar-navigation-with-fab-8b962bb55013](https://medium.com/coding-with-flutter/flutter-bottomappbar-navigation-with-fab-8b962bb55013)
* [https://iirokrankka.com/2017/09/04/clipping-widgets-with-bezier-curves-in-flutter/](https://iirokrankka.com/2017/09/04/clipping-widgets-with-bezier-curves-in-flutter/)
* [https://docs.google.com/document/d/e/2PACX-1vRVPWGtR85bawGynRSWzYTKgQtqrxCnxXCKC5xM9ab3IvtRHueku4rRIuJ4TbedzyMz2oy2pkzM71-_/pub](https://docs.google.com/document/d/e/2PACX-1vRVPWGtR85bawGynRSWzYTKgQtqrxCnxXCKC5xM9ab3IvtRHueku4rRIuJ4TbedzyMz2oy2pkzM71-_/pub)