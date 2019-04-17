---
layout: post
title: "自定义Property属性动画"
header_image: /assets/img/2016-03-06-20.jpg
description: ""
category: "customlayout"
tags: [android]
---
{% include JB/setup %}
![img](/assets/img/2016-03-06-20.jpg)

## 前言
	
	代码获取
	git clone https://github.com/avenwu/support.git 

在Android中动画的实现有许多不同选择，本文将扩展FrameLayout为其添加背景动画；

针对某个view做动画比较方便，这里通过自定义的属性来为一个容器类布局添加背景动画；

![Property动画](/assets/property_animation.gif)

## 思路
1. 动画的原理本质就是修改属性值，然后根据新的值进行绘制；
2. 采用ObjectAnimator，其内部实现了对值再给定时间内的变化处理；
3. 定义代表缩放圆圈的半径属性，刷新视图；

## 实战
根据需要，先定义float型半径mRippleRadius，并提供相应地setter、getter

{% highlight java %}
    private float mRippleRadius;
    private float getRadius() {
        return mRippleRadius;
    }

    private void setRadius(float radius) {
        this.mRippleRadius = radius;
    }

{% endhighlight %}


添加Property

{% highlight java %}
    Property<BreathingDelegate, Float> mRadiusProperty = new Property<BreathingDelegate, Float>(Float.class, "mRippleRadius") {
        @Override
        public Float get(BreathingDelegate object) {
            return object.getRadius();
        }

        @Override
        public void set(BreathingDelegate object, Float value) {
            object.setRadius(value);
        }
    };

{% endhighlight %}

现在利用ObjectAnimator，实现mRippleRadius的变化，需要注意的是再每次值变化后，这里手动调用的invalidate保证视图会刷新；

{% highlight java %}

            ObjectAnimator animator = ObjectAnimator.ofFloat(this, mRadiusProperty, mRippleRadius, mEndRadius);
        animator.setDuration(mDuration);
        animator.setRepeatCount(ValueAnimator.INFINITE);
        animator.setRepeatMode(ValueAnimator.REVERSE);
        animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                mTarget.invalidate();
            }
        });        
{% endhighlight %}


最后重载绘制方法，为容器绘制我们希望看到的背景
{% highlight java %}
    public void onDraw(Canvas canvas) {
        mRippleRect.set(mBorderRect.centerX() - mRippleRadius, mBorderRect.centerY() - mRippleRadius,
                mBorderRect.centerX() + mRippleRadius, mBorderRect.centerY() + mRippleRadius);
        canvas.drawOval(mRippleRect, mPaint);
        Log.d("BreathingLayout", "onDraw=" + mRippleRect.toString());
    }
{% endhighlight %}

## 结语
ObjectAnimator/ValueAnimator不单单可以用在常规缩放，位移动画中，也可于再自定义的属性，以及在很多需要线性变化的地方。

---


