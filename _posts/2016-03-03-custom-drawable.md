---
layout: post
title: "如何写一个能解决问题的Drawable"
description: "自定义android drawable，解决实际问题，如边框线"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-03-01.jpg
keywords: "自定义Drawable" 
tags: [android, drawable]
---
{% include JB/setup %}

![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-03-01.jpg)

## 前言
通过xml写shape实现规则的几何图形，相信列位都不陌生。在实际开发中，有时候光靠xml实现的形状还是不能尽如人意；  

	穷则变，变则通

举个很简单的例子，九宫格的分割线，tab的边框线;  
shape的的常用属性solid，stroke,width,pading等等，实现起来的图形往往都是规则的，对称的，因此诸如tab的边框线，如果用shape实现
那么必然会导致相邻的边线重复，导致这条边界看起来比其他先要粗，如下如：  
![1-1]({{ site.baseurl }}/assets/images/rect_with_shape_xml.jpg)

当然这种情况解决的办法很多，比如直接找美工做张图，这样可行，但是不灵活，如果是动态生成不同个数的tab，则需要打散图片为多张；  

类似的问题也会出现在其他拼接视图的地方；  
那么有没有什么办法可以不让美工作图，直接布局或者代码实现？  

	答案是自定义Drawable，用代码绘制任意我们需要的几何图形；  

## 案例分析
仍然是上述的例子，我希望他的最终效果是每条边一样粗；  

![1-2]({{ site.baseurl }}/assets/images/rect_final_effect.png)

首先把目标效果图做一下分解，大致是三块：左边带圆角，中间无圆角，右侧圆角矩形；  
同时三个矩形的四边宽度是不同的，这样拼接在一起才能达到理想效果； 
 
![1-3]({{ site.baseurl }}/assets/images/rect_parts.png)

现在可以通过作图工具绘制出三种类型的形状；但是我们可以通过简单的画笔直接绘制出灵活性更大的图形；

## 自定义Drawable
实际上，平时我们用xml写的shape最终系统会通过接口函数解析为ShapeDrawable等等具体的Drawable；  
打开ShapeDrawable可以发现，其内部非常简洁，主要就是一些画笔，样式，和常量的保存，自己写一个简单的drawable基本上知道这些就可以了，至于ConstantState可以跳过，因为系统框架的原因，我们没有可能用xml来自定义drawable，只能是代码直接生成；  

![1-4]({{ site.baseurl }}/assets/images/shapedrawable_methods.jpg)

通过分析ShapeDrawable源码，可以发现，最终的绘制也是在onDraw内完成，而需要绘制的内容在bound变化时同时修改；  
回到我们的的需求，要绘制上面的三种图片，可以利用Path，在canvas行绘制我们指定的Path，而三种图形的path基本是一样的，区别在于是否有圆角，每条边线的宽度；

* 圆角问题可以arc，及在Path中适当的位置指定一个圆弧路径；  
* 边线问题，path不能直接设置，但是path可以决定需要绘制的图形的位置；


		因此，我们可以巧妙利用path绘制线条时指定位置来绘制不同粗细的路径

现在我们一第一张图为例，吧图放到坐标系内，分析出各个关键点的坐标值：  

![1-4]({{ site.baseurl }}/assets/images/rect_point_x_y.jpg)

* 首先绘制A-B这一段，绘制弧线这里用的是arc，需要一个弧线所在的矩形框，其实角度，弧形角度大小

```
RectF topLeftRect = new RectF(strokeWidth, strokeWidth, width, width);
path.arcTo(topLeftRect, 180, 90);
```

* 绘制B-C直线段，直接line

```
path.lineTo(r.width(), strokeWidth);
```
* 绘制C-D直线段

```
path.lineTo(r.width(), r.height() - strokeWidth);
```
* 绘制D-E直线段

```
path.lineTo(width + strokeWidth, r.height() - strokeWidth);
```
* 绘制E-F弧形

```
RectF BottomLeftRect = new RectF(strokeWidth, r.height() - width - strokeWidth, width, r.height() - strokeWidth);
path.arcTo(BottomLeftRect, 90, 90);
```

最后关闭path，在onDraw内直接绘制即可：  

```
@Override
public void draw(Canvas canvas) {
    canvas.drawPath(mPath, mPaint);
}
```

## 小结
实际上自己绘制一个drawable不是很麻烦，但是关键看怎么样满足需求，过于复杂的图形就没必要手动编码实现，君子善假于物就是这个道理。
