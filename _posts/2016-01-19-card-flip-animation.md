---
layout: post
title: "卡片翻转运动分析及翻转优化"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-06.jpg
keywords: "卡片翻转 动画"
description: "card flip animation"
category: 
tags: [android,优化]
---
{% include JB/setup %}

![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-06.jpg)

## 前言

卡片翻转是比较常见的过场动画，一般可以通过属性动画实现，比如横向翻转则实际上是绕Y轴旋转；
	
![device-2016-01-19-162559.mp4.gif]({{ site.baseurl }}/assets/images/device-2016-01-19-162559.mp4.gif)  

android developer Training中有一章节提到了翻转动画的实现，看着效果也还行，但是实际使用的时候会发现效果比较生硬，特别是全屏翻转，在翻转接近90时画面比较惨烈,有种扑面而来的赶脚；  


![device-2016-01-19-163731.mp4.gif]({{ site.baseurl }}/assets/images/device-2016-01-19-163731.mp4.gif)


原因在于二维平面纯粹的绕轴旋转本身在视觉上不能完全模拟出现实中3d的翻转效果，因此，需要在翻转过程中加上“景深”效果；

## 运动分析
在绘图中我们知道，平面绘画产生立体效果主要依靠“光影”和“近大远小”的几何原理；这里我们可以利用“近大远小”这一点做文章。

先分析一下翻转的几个状态；  

[]()

  
    
    

* 当视图项内翻转时，视觉上，视图是向远处运动，在翻转到90°,即垂直于屏幕时达到最远，在这一段内只需根据比例不断缩小视图；
* 90°之后，将作为背面的视图开始由远至近翻转，在翻转到0°，即水平于屏幕时达到最近，在这一段时间内，正好和上述是相反的，只需要就根据比例不断放大视图；

## 优化实现
在Android上，有很多类型的动画，不同技术点可以不同方法，这里用比较直观，同时也是被采用最多的实现方案Camera和Matrix;

camera对应绘制过程中的视觉窗口，matrix是动画变化的矩阵；  


{% highlight java %}

        @Override
        protected void applyTransformation(float interpolatedTime, Transformation t) {
            float degree = mFrom + (mTo - mFrom) * interpolatedTime;
            float scale = mScaleDown ? (1 - (1 - SCALE_DEFAULT) * interpolatedTime) :
                    (SCALE_DEFAULT + (1 - SCALE_DEFAULT) * interpolatedTime);
            float alpha = mScaleDown ? (1 - (1 - ALPHA_DEFAULT) * interpolatedTime) :
                    (ALPHA_DEFAULT + (1 - ALPHA_DEFAULT) * interpolatedTime);
            t.setAlpha(alpha);

            final Matrix matrix = t.getMatrix();
            mCamera.save();
            mCamera.rotateY(degree);
            mCamera.getMatrix(matrix);
            mCamera.restore();

            matrix.preTranslate(-mCenterX, -mCenterY);
            matrix.postTranslate(mCenterX, mCenterY);
            matrix.preScale(scale, scale, mCenterX, mCenterY);
        }
{% endhighlight %}

* 首先当前动画执行的状态，计算翻转角度，缩放值，透明值；
* 为摄像机设置翻转角度；
* 为实现相对位置的旋转效果，为matrix设置matrix.preTranslate和matrix.postTranslate；

关于matrix.postTranslate的作用可以查阅结尾的参考文献[http://www.satyakomatineni.com/akc/display?url=displaynoteimpurl&ownerUserId=satya&reportId=2898](http://www.satyakomatineni.com/akc/display?url=displaynoteimpurl&ownerUserId=satya&reportId=2898)
下面这两张图分别是正常效果和不设置matrix偏移的效果, 应该还是比较直观的：
![device-2016-01-19-162559.mp4.gif]({{ site.baseurl }}/assets/images/device-2016-01-19-162559.mp4.gif) 

![device-2016-01-19-162713.mp4.gif]({{ site.baseurl }}/assets/images/device-2016-01-19-162713.mp4.gif) 

核心的原理实现就是上述applyTransformation内的对camera和matrix的操作；
完整代码可以这里查阅：

	https://github.com/avenwu/support
	

## Reference
* [http://developer.android.com/training/animation/cardflip.html](http://developer.android.com/training/animation/cardflip.html)
* [https://github.com/saulpower/Android-3D-Flip-Transition](https://github.com/saulpower/Android-3D-Flip-Transition)
* [https://github.com/genzeb/flip](https://github.com/genzeb/flip)
* [http://www.satyakomatineni.com/akc/display?url=displaynoteimpurl&ownerUserId=satya&reportId=2898](http://www.satyakomatineni.com/akc/display?url=displaynoteimpurl&ownerUserId=satya&reportId=2898)
