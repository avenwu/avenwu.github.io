---
layout: post
title: "基于SlidePanelLayout实现ResideMenu"
description: ""
category: "customlayout"
tags: [android,自定义Layout]
---
{% include JB/setup %}

14年的时候曾经在csdn上看到过一篇介绍Android中ResideMenu实现的推荐项目，也分析了一把，详见[Android控件源码分析--AndroidResideMenu菜单](http://www.cnblogs.com/avenwu/p/3482199.html)
，那时候QQ还没有redisemenu的效果。效果还是不错的，但是有一个确定就是不支持平滑拖动时对菜单，容器的控制，只是简单的动画实现。  
项目灵感据说是来自于dribble网站上的两个交互设计原型：

![dribbble_1x.png](http://7u2jir.com1.z0.glb.clouddn.com/dribbble_1x.png)
![social_feed_ios7.gif](http://7u2jir.com1.z0.glb.clouddn.com/social_feed_ios7.gif)

**相关链接如下**：  
[1116265-Instasave-iPhone-App](https://dribbble.com/shots/1116265-Instasave-iPhone-App)
[https://dribbble.com/shots/1114754-Social-Feed-iOS7](https://dribbble.com/shots/1114754-Social-Feed-iOS7)

现在QQ的侧滑ResideMenu效果和上面非常类似，同时支持平滑拖动的处理；

网络上也有一些高仿QQ策划效果的实现，比如基于HorizonalScrollView的，基于SlideMenu修改的，都是不错的思路；本文打算基于android 官方v4包进行扩展，还是更偏向使用官方的东西：）

下面是qq效果和自定义的效果：  

![qq_residemenu.gif](http://7u2jir.com1.z0.glb.clouddn.com/qq_residemenu.gif)
![custom_residemenu.gif](http://7u2jir.com1.z0.glb.clouddn.com/custom_residemenu.gif)

###实现
基于v4扩展包的SlidePanelLayout可以比较方便就实现需要的效果。
核心在于对互动状态的处理，SlidePanelLayout有一个setPanelSlideListener监听，类似于onScroll，通过这个回调可以很容易知道当前欢动的百分比，从而改变视图的位置，实现动画中效果。

需要注意的是这里用到了level 11以后的方法，所以如果要兼容11以前的设备可以用nineoldedroid做修改。


{% highlight java %}
package net.avenwu.support.widget;


import android.annotation.TargetApi;
import android.content.Context;
import android.graphics.Canvas;
import android.graphics.Color;
import android.os.Build;
import android.support.v4.widget.SlidingPaneLayout;
import android.util.AttributeSet;
import android.view.View;

/**
 * Support level 11 and later;
 * TODO use nineolddroid for devices under level 11
 * <p/>
 * Created by chaobin on 2/18/15.
 */
@TargetApi(11)
public class CustomSlidePanelLayout extends SlidingPaneLayout {
    protected View mMenuPanel;
    protected float mSlideOffset;
    protected boolean isCustomable = false;

    public CustomSlidePanelLayout(Context context) {
        this(context, null);
    }

    public CustomSlidePanelLayout(Context context, AttributeSet attrs) {
        this(context, attrs, -1);
    }

    public CustomSlidePanelLayout(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB) {
            isCustomable = true;
            setPanelSlideListener(new SimplePanelSlideListener() {
                @Override
                public void onPanelSlide(View panel, float slideOffset) {
                    mSlideOffset = slideOffset;
                    if (mMenuPanel == null) {
                        final int childCount = getChildCount();
                        for (int i = 0; i < childCount; i++) {
                            final View child = getChildAt(i);
                            if (child != panel) {
                                mMenuPanel = child;
                                break;
                            }
                        }
                    }
                    final float scaleLeft = 1 - 0.3f * (1 - slideOffset);
                    mMenuPanel.setPivotX(-0.3f * mMenuPanel.getWidth());
                    mMenuPanel.setPivotY(mMenuPanel.getHeight() / 2f);
                    mMenuPanel.setScaleX(scaleLeft);
                    mMenuPanel.setScaleY(scaleLeft);

                    final float scale = 1 - 0.2f * slideOffset;
                    panel.setPivotX(0);
                    panel.setPivotY(panel.getHeight() / 2.0f);
                    panel.setScaleX(scale);
                    panel.setScaleY(scale);
                }
            });
            setSliderFadeColor(0);
        }

    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        if (isCustomable) {
            dimOnForeground(canvas);
        }
    }

    @Override
    protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        boolean result = super.drawChild(canvas, child, drawingTime);
        if (isCustomable && child == mMenuPanel) {
            dimOnForeground(canvas);
        }
        return result;
    }

    /**
     * dim the view
     *
     * @param canvas
     */
    private void dimOnForeground(Canvas canvas) {
        canvas.drawColor(Color.argb((int) (0xff * (1 - mSlideOffset)), 0, 0, 0));
    }
}

{% endhighlight %}

####相关链接
1. [https://github.com/romaonthego/RESideMenu](https://github.com/romaonthego/RESideMenu)
2. [https://github.com/SpecialCyCi/AndroidResideMenu](https://github.com/SpecialCyCi/AndroidResideMenu)