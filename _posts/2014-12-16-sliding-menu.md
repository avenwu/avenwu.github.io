---
layout: post
title: "自定义侧滑菜单"
description: "自定义一个简易的侧滑菜单，类似于DrawerLayout/SlidingMenu"
category: "customlayout"
tags: [自定义Layout,android]
---
{% include JB/setup %}

无图无真相，[完整代码](https://github.com/avenwu/support/blob/master/support/src/main/java/net/avenwu/support/widget/DrawerFrame.java)  

![demo效果图](http://7u2jir.com1.z0.glb.clouddn.com/drawermenu.gif)

####思路

从前两年侧滑菜单出现到火热，到现在成为一种很常见的交互布局，作为开发者的我们其实选择非常多了，既有开源的也有官方的。
	
* Android support v4扩展包的[DrawerLayout](https://developer.android.com/reference/android/support/v4/widget/DrawerLayout.html)
* Android support v4扩展包的[SlidingPaneLayout](https://developer.android.com/reference/android/support/v4/widget/SlidingPaneLayout.html)
* 比较知名的三方开源库[SlidingMenu](https://github.com/jfeinstein10/SlidingMenu)
* 其他...

这些控件的底层使用的技术实际上是类似的，为了更深入的悉知这些轮子是怎么造的，本文将着手实现一个简易的侧滑控件。

####设计思路
首先可以一起分析一下，实现一个最基础的菜单需要解决那些技术点。

* 界面分为菜单区和内容区，通过滑动显示、隐藏菜单
* 手势分发处理
* 根据滑动停止后，根据位置自动完成显示、隐藏操作

####实现细节
根据前面提到的几个技术点，现在开始逐一处理。

#####区域划为
这里选择FrameLayout作为基类，这样可以免去处理菜单视图和内容视图的层级关。

{% highlight java %}		
main = new FrameLayout(getContext());
main.setId(R.id.main);
main.setLayoutParams(new FrameLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT,
                ViewGroup.LayoutParams.MATCH_PARENT));
main.setBackgroundColor(getResources().getColor(android.R.color.holo_blue_bright));
addView(main);

left = new FrameLayout(getContext());
left.setId(R.id.menu);
left.setLayoutParams(new FrameLayout.LayoutParams(MENU_WIDTH, ViewGroup.LayoutParams.MATCH_PARENT));
addView(left);
{% endhighlight %}

#####手势分发
手势处理包括touch event分发和消耗，在onInterceptTouchEvent中可以简单判断当前是否处于菜单滑动状态，是的话拦截后续的手势。

{% highlight java %}		
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
        if (!mSlidable) return false;
        final int action = ev.getAction();
        if (action != MotionEvent.ACTION_DOWN && isSliding) return true;
        switch (action) {
                case MotionEvent.ACTION_CANCEL:
                case MotionEvent.ACTION_UP:
                        return false;
                case MotionEvent.ACTION_MOVE:
                        if (Math.abs(ev.getX() - mSrcX) > touchSlop) {
                                mSrcX = ev.getX();
                                isSliding = true;
                        }
                        break;
                case MotionEvent.ACTION_DOWN:
                        isSliding = false;
                        mSrcX = ev.getX();
                        break;
        }
        return isSliding;
}
{% endhighlight %}

#####手势处理
现在需要处理菜单的滑动位置，在滑动过程中，MotionEvent.ACTION_MOVE不断被触发，所以可以在这里改变菜单view的位置，此处利用Scroller负责位置的变化；MotionEvent.ACTION_UP中判断手势抬起时的菜单位置状态，如果滑动位置已经达到菜单宽度的1/2,那么认为菜单需要继续打开，反之收起。

{% highlight java %}		
@Override
public boolean onTouchEvent(MotionEvent event) {
        d("UIView", "event:" + event.toString());
        final int action = event.getAction();
        switch (action) {
                case MotionEvent.ACTION_DOWN:
                        mSrcX = event.getX();
                        break;
                case MotionEvent.ACTION_MOVE:
                        int oldx = left.getScrollX();
                        float dx = mSrcX - event.getX();
                        if (dx != 0) {
                                left.setVisibility(VISIBLE);
                                float x = oldx + dx;
                                d("onTouchEvent", "move, oldx=" + oldx + ", dx=" + dx + ", left=" +
                                                left.getLeft() + ", right=" + left.getRight() + ", getX=" + event.getX() + ", mSrcX=" + mSrcX);
                                d("onTouchEvent", "before x=" + x);
                                if (MENU_WIDTH < x) {
                                        x = MENU_WIDTH;
                                }
                                if (0 > x) {
                                        x = 0;
                                }
                                d("onTouchEvent", "after x=" + x);
                                left.scrollTo((int) x, 0);
                                mSrcX = event.getX();
                        }
                        break;
                case MotionEvent.ACTION_UP:
                        int currentX = left.getScrollX();
                        if (currentX + mSrcX - event.getX() >= MENU_WIDTH / 2.0) {
                                int duration = (int) (Math.abs(MENU_WIDTH - currentX + 0.5f) / MENU_WIDTH * 1000);
                                scroller.startScroll(currentX, 0, MENU_WIDTH - currentX, 0, duration);
                                invalidate();
                        } else {
                                int duration = (int) (Math.abs(currentX + 0.5f) / MENU_WIDTH * 1000);
                                scroller.startScroll(currentX, 0, 0 - currentX, 0, duration);
                                invalidate();
                        }
                        break;
                case MotionEvent.ACTION_CANCEL:
                        break;
        }
        return true;
}
{% endhighlight %}

#####初始化视图位置
除了手势问题，还需要将视图在容器中初始化位置，这里需要复写onLayout，由于只有两个子view（菜单区，内容区），默认菜单处于关闭状态。

{% highlight java %}
@Override
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        int count = getChildCount();
        for (int i = 0; i < count; i++) {
                View child = getChildAt(i);
                if (child.getVisibility() == GONE) continue;
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                if (child.getId() == R.id.menu) {
                        final int childWidth = child.getMeasuredWidth();
                        final int childHeight = child.getMeasuredHeight();
                        int childLeft = 0;
                        child.layout(childLeft, lp.topMargin, childLeft + childWidth, lp.topMargin + childHeight);
                        child.scrollTo(MENU_WIDTH, 0);
                } else if (child.getId() == R.id.main) {
                        child.layout(0, 0, child.getMeasuredWidth(), child.getMeasuredHeight());
                }
        }
}
{% endhighlight %}

#####平滑滚动
前面已经提到改变菜单位置是利用了Scroller,主要是考虑到平滑滚动问题。

{% highlight java %}
@Override
public void computeScroll() {
        d("computeScroll", "computeScroll");
        if (scroller.computeScrollOffset()) {
                int oldx = left.getScrollX();
                int x = scroller.getCurrX();
                d("computeScroll", "try scroll, oldx=" + oldx + ", x=" + x);
                if (oldx != x) {
                        //this can only effect on the content view inside of left
                        left.scrollTo(x, 0);
                        left.invalidate();
                }
                invalidate();
        } else {
                scroller.abortAnimation();
        }
}
{% endhighlight %}

###小结
至此自定义一个简易的侧滑菜单涉及的主要技术点都解决了，其他细节可以看完整代码