---
layout: post
title: "让你的AppBar凹凸有致"
description: "凸起的AppBar设计和持续优化"
header_image: https://github.com/hacktons/convex_bottom_bar/raw/master/doc/preview.png
keywords: ""
tags: []
---
{% include JB/setup %}
![img](https://github.com/hacktons/convex_bottom_bar/raw/master/doc/preview.png)

* 目录
{:toc #markdown-toc}

## 背景

在[Flutter凸起导航栏优化](http://blog.hacktons.cn/2019/09/02/diy-navigation-bar/)一文中，笔者分享了实现凸起的导航栏效果的思路。后面收到了网友的邮件咨询，希望了解具体的代码实现细节。

本文针对凸起导航的实现和社区其他项目的解决方案做了分析讲解，同时提供了项目开源版本，喜欢的小伙伴可以前去[start/fork](https://github.com/hacktons/convex_bottom_bar)

> [https://github.com/hacktons/convex_bottom_bar](https://github.com/hacktons/convex_bottom_bar)

## 视觉设计
我们要实现的效果大致如下，一个普通的Tab导航，中间Tab有一个凸起弧度

![](https://github.com/hacktons/convex_bottom_bar/raw/master/doc/Screenshot_1571041912.png)

对视觉稿进行分解一下，细化出我们需要实现的功能点
>* TAB导航栏
* TAB导航栏设置，颜色，高度
* TAB的高亮设置和图文支持
* 中间TAB凸起，弧度，大小
* 点击事件感知

## 贴图大法
贴图法是最实现成本最低的一种方案，我们只需要把凸起作为TAB背景的一部分，固化在背景图上即可，同理投影也可以绘制在背景图上。
贴图有一个弊端：屏幕适配不佳。

试想一下终端设备分辨率那么多，虽然我们可以针对主流的设备提供不同分辨率的图片，甚至横竖屏也要额外考虑下；对于位图，还要注意凸起部分不能拉伸失真。

```dart
return Stack(
  overflow: Overflow.visible,
  alignment: Alignment.bottomCenter,
  children: <Widget>[
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

## 定制化BottomAppBar
仔细观察UI效果，如果适当的分解，可以把凸起的Tab作为一个floatingActionButton，这个按钮的位置正好固定在AppBar的中心。然后通过绘制的方式，组合凸起按钮和AppBar。也就是我们在[Flutter凸起导航栏优化](http://blog.hacktons.cn/2019/09/02/diy-navigation-bar/)里面实现的方案。

相关实现可以参考：[适配BottomAppBar](https://github.com/hacktons/convex_bottom_bar/blob/f-polish-sample/README-zh.md)

通过扩展的FAB，我们在使用的时候有几个核心步骤：
* 添加FAB按钮
* 使FAB居中展示
* 添加AppBar按钮

在效果上没有问题，使用也方便，但是如果作为对外提供的插件，它有一个“缺点”：
> 凸起控件实现的粒度比较粗，需要开发者控制的属性多，步骤要求严格。

```dart
Scaffold(
  appBar: AppBar(
    title: const Text('Default ConvexAppBar'),
  ),
  body: Center(
    child: Text('TAB $_selectedIndex', style: TextStyle(fontSize: 20)),
  ),
  floatingActionButton: ConvexAppBar.fab(
    text: 'Publish',
    active: _selectedIndex == INDEX_PUBLISH,
    activeColor: ACTIVE_COLOR,
    color: NORMAL_COLOR,
    onTap: () => onTabSelected(INDEX_PUBLISH),
  ),
  floatingActionButtonLocation: ConvexAppBar.centerDocked,
  bottomNavigationBar: ConvexAppBar(
    items: TAB_ITEMS,
    index: _selectedIndex,
    activeColor: ACTIVE_COLOR,
    color: NORMAL_COLOR,
    onTap: onTabSelected,
  ),
)
```

上面的痛点转换成需求，其实如果能实现一个独立ConvexAppBar控件，完全控制FAB和位置就更好了

## 手工打造ConvexAppBar
我们在回顾下上面的实现方法，造成封装粒度的原因本质在于：
> 我们依赖了floatingActionButton和他的锚点位置，这个属性天生就是和bottomNavigationBar同级的，如果我们要强行抹平，就需要解除依赖关系。

ConvexAppBar项目就是基于这个目标产生的，实现的思路是自定义并绘制凸起曲线。在分析之前先看一下最终的效果，支持内置样式的调整:
![](https://github.com/hacktons/convex_bottom_bar/blob/master/doc/appbar-theming.png)

### UI模型优化

类似贴图法，如果用自定义绘制，也需要对AppBar进行合理分拆，我们把曲线单独作为一层绘制，通过叠加按钮层，凸起按钮，最终达到目标效果。
![](/assets/images/appbar-layer.png)

到现在我们已经有几种凸起tab的实现思路

| 方案       | 基本原理                             | 优缺点                 | 实现复杂度 |
| ---------- | ------------------------------------ | ---------------------- | ---------- |
| 一、贴图法    | 通过切图叠加实现                     | 适配复杂，效果不是最佳 | 低         |
| 二、FAB绘制    | 利用floatingActionButton的个性化绘制 | 效果好，有依赖配置     | 中         |
| 三、自定义绘制 | 自行绘制凸起曲线                     | 效果好，无依赖         | 高         |

### 画笔基本操作
在自定义过程中，需要了解一些画笔基本操作，比如外投影/内投影，弧线，半圆等处理。
* 投影： canvas.drawShadow可以绘制投影，但是没法控制方向，利用MaskFilter也可以绘制投影效果
* 弧线：path.arcTo，canvas.drawArc, 半圆和弧度依赖角度的控制
* 路径绘制： canvas.drawPath， Path可以自行构造出任意的几何形状

### 坐标系
画布的坐标系，左上角圆点(0,0)，右下角(width,height)

![](/assets/images/curve-1.png)

### 实例分析一，凸起实现
在绘制曲线的时候，根据实际凸起的幅度不同可以采用近似求解，比如如果凸起是个半圆，我们可以用分解为下图：
![](/assets/images/curve-1.png)

主体半圆的左右各有一个弧度，弧度通过两个小的矩形来定位并绘制，当半圆和矩形的倍数足够大时，C1点和C2点将"重合"，因此可以近似的把这个图的作为绘制凸起的公式。

![](/assets/images/Screenshot_1571656525.png)

示意图中居中绘制了一个半圆曲线，核心逻辑为在限定大小的控件内，基于坐标计算绘制路径
```dart
SizedBox(
  width: 90,
  height: 90,
  child: CustomPaint(
    painter: ConvexPainter(Colors.redAccent),
  ),
),
```

可以看到tab部分的半圆凸起效果还是很逼真的，但是如果放大细节查看，或者将半圆高度变成非半圆，那么近似求解就会出现肉眼可见的瑕疵。

为了凸显问题，我们把颜色调一下，并放大细节

![](/assets/images/Screenshot_1571657237.png)

### 实例分二析，凸起优化
因此，如果要更好的细节效果，同时允许设置凸起幅度，那么我们就不能采用近似求解的方式。
计算曲线可以使用贝塞尔曲线, 在计算notch的文章中有详细的数理推算。

![](/assets/images/finding-p1.png)

计算P1,P2利用了圆的切线，相当于已知P0和圆心，半径，解救P1,P2的坐标。感兴趣的小伙伴可以自行推理，这里我们直接利用公式。

为了方便主体个性化，增加一些属性配置：

![](https://github.com/hacktons/convex_bottom_bar/raw/master/doc/appbar-theming.png)

## 小结
通过自定义凸起Tab，运用到自定义CustomPaint控件和画笔的使用，以及针对不同场景的近似求解，虽然最后我们没有采用，但是这种方案也不失为一种简化计算的方法。

## 参考
* [Computing a smooth circular notch](https://goo.gl/Ufzrqn)
