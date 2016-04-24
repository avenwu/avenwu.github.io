---
layout: post
title: "Make your OverScrollLayout"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-04-24-01.jpg
keywords: ""
tags: [android]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-04-24-01.jpg)

## 前言
手势处理在自定义控件中随处可见，诸如下拉，侧滑，加载更多等等，本文将利用手势相关的知识，来实现一个类似微信文章中过渡滑动交互。

下滑的处理在很早之前也写过几篇分别是简单的下拉，侧滑，原理基本类似，主要依赖的是offsetTopBotom,offsetLeftRight,Scroller；

在处理OverScroll的时候需要正确决定手势是否被拦截用于，滑动视图，或者分发至系统时间响应，根据内容区视图是否能滑动也有细节上的处理，这一块可以参照SwpieRefreshLayout的处理，即提供一个检测方法canChildScroolUp，针对内容view的类型单独处理;

下面是最终实现的效果：  
![WebView]({{ site.baseurl }}/assets/images/device-2016-04-24-102407.gif) ![RecyclerView]({{ site.baseurl }}/assets/images/device-2016-04-24-102722.gif)

## 控件定义
首先考虑一下，这个效果的UI组成，当滑动到顶部时可以继续滑动，展示底部的视图；因此滑动主要解决的是视图在Y轴上的偏移：

![Layout]({{ site.baseurl}}/assets/images/over-scroll-layout.jpg)

* 区分内容视图和背景视图
允许视图内包含一个背景View和一个内容View，为便于理解，参照FrameLayout，xml内第一个的view作为底视图，第二个作为内容视图
对子View的关系约定，可以放在合适的地方，比如inflate，或者第一次measure，layout时

{% highlight java %}
if (count > 1) {
    mForegroundView = getChildAt(1);
    mBackgroundView = getChildAt(0);
} else {
    mForegroundView = getChildAt(0);
}
{% endhighlight %}

* 视图排版
初始化视图，后需要根据规范对View的measure，layout响应处理

{% highlight java %}
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        int count = getChildCount();
        log("onMeasure: count=" + count);
        if (count > 1) {
            mForegroundView = getChildAt(1);
            mBackgroundView = getChildAt(0);
        } else {
            mForegroundView = getChildAt(0);
        }
        if (mBackgroundView != null) {
            measureChildWithMargins(mBackgroundView, widthMeasureSpec, 0, heightMeasureSpec, 0);
        }
        if (mForegroundView != null) {
            measureChildWithMargins(mForegroundView, widthMeasureSpec, 0, heightMeasureSpec, 0);
            mHeightLimitMax = (int) (mForegroundView.getMeasuredHeight() * mMaxOverPercent);
            if (mForegroundView instanceof ScrollIntercept) {
                ((ScrollIntercept) mForegroundView).attach(this);
            }
        }
        if (mForegroundView != null && mBackgroundView != null) {
            mForegroundView.bringToFront();
        }
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        if (mBackgroundView != null) {
            MarginLayoutParams layoutParams = (MarginLayoutParams) mBackgroundView.getLayoutParams();
            int left = layoutParams.leftMargin + getPaddingLeft();
            int top = layoutParams.topMargin + getPaddingTop();
            int right = left + mBackgroundView.getMeasuredWidth();
            int bottom = top + mBackgroundView.getMeasuredHeight();
            mBackgroundView.layout(left, top, right, bottom);
            log("onLayout, mBackgroudView = " + mBackgroundView.toString() +
                    ", l=" + left + ",t=" + top + ", r=" + right + ",b=" + bottom);
        }
        if (mForegroundView != null) {
            MarginLayoutParams layoutParams = (MarginLayoutParams) mForegroundView.getLayoutParams();
            int left = layoutParams.leftMargin + getPaddingLeft();
            int top = layoutParams.topMargin + getPaddingTop();
            int right = left + mForegroundView.getMeasuredWidth();
            int bottom = top + mForegroundView.getMeasuredHeight();
            mForegroundView.layout(left, top, right, bottom);
            log("onLayout, mForegroundView = " + mForegroundView.toString() +
                    ", l=" + left + ",t=" + top + ", r=" + right + ",b=" + bottom);
        }
    }
{% endhighlight %}

* 重载onInterceptTouchEvent
当手势在滑动且是上下滑动，超过touchslop时，开始拦截；

{% highlight java %}
    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        log("onInterceptTouchEvent:" + ev.toString());
        if (!isEnabled() || canChildScrollUp()) {
            //make child layout to determine the scroll state
            if (mForegroundView instanceof ScrollIntercept &&
                    (ev.getAction() == MotionEvent.ACTION_CANCEL || ev.getAction() == MotionEvent.ACTION_UP)) {
                log("try tryScrollToTop");
                ((ScrollIntercept) mForegroundView).tryScrollToTop(this);
            }
            return false;
        }
        final int action = ev.getAction();
        if (action != MotionEvent.ACTION_DOWN && isSliding) return true;
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                isSliding = false;
                mX = ev.getX();
                mY = ev.getY();
                break;
            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP:
                isSliding = false;
                break;
            case MotionEvent.ACTION_MOVE:
                float dy = ev.getY() - mY;
                log("onInterceptTouchEvent, canChildScrollUp=" + canChildScrollUp());
                if (dy > mTouchSlop && !isSliding) {
                    mY = ev.getY();
                    mX = ev.getX();
                    isSliding = true;
                    log("dy=" + dy);
                }
                break;

        }
        log("onInterceptTouchEvent, isSliding=" + isSliding);
        return isSliding;
    }
{% endhighlight %}

* 重载onTouch
这里可以开始根据move的具体值，调整view的offset
{% highlight java %}
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        log("onTouchEvent:" + event.toString());
        if (!isEnabled() || canChildScrollUp()) {
            // Fail fast if we're not in a state where a swipe is possible
            return false;
        }
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                mX = event.getX();
                mY = event.getY();
                isSliding = false;
                if (!mScroller.isFinished()) {
                    mScroller.abortAnimation();
                }
            case MotionEvent.ACTION_MOVE:
                float offsetX = event.getX() - mX;
                float offsetY = event.getY() - mY;
                log("move, offsetX=" + offsetX + ", offsetY=" + offsetY + ", mTouchSlop=" + mTouchSlop);
                if (isSliding) {
                    if (offsetY > 0) {
                        scrollY((int) offsetY);
                        mX = event.getX();
                        mY = event.getY();
                    } else {
                        return false;
                    }
                }
                break;
            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP:
                if (!mScroller.isFinished()) {
                    mScroller.abortAnimation();
                }
                int currentY = mForegroundView.getTop();
                int gap = (int) (mForegroundView.getHeight() * mOverPercent);
                if (currentY + event.getY() - mY >= gap / 2) {
                    int duration = (int) (Math.abs(0 - currentY + 0.5f) / mForegroundView.getHeight() * 1000);
                    log("duration=" + duration);
                    mScroller.startScroll(0, currentY, 0, 0 - currentY, duration);
                } else {
                    int duration = (int) (Math.abs(-gap - currentY + 0.5f) / mForegroundView.getHeight() * 1000);
                    log("duration=" + duration);
                    mScroller.startScroll(0, currentY, 0, -gap - currentY, duration);
                }
                isSliding = false;
                invalidate();
                return false;
        }
        return true;
    }
{% endhighlight %}

* 支持WebView
需要注意的是WebView在JellyBean之后内部实现有了很大变化，参考其源码实现可以证实，大量的具体逻辑都已经委托个了一个Provider实现类，这直接导致webview无法和其他view一样检测出其滑动位置；因此为了兼容WebView，需要在WebView内的OverScroll回调内手动再次处理offset，这一点实际上时参考了ChrisBean的Pull2Refresh的实现；

{% highlight java %}
    @Override
    public void tryScrollToTop(ScrollOver layout) {
        log("tryScrollToTop:" + mScrolledY);
        if (mScrolledY != 0) {
            mLayout.smooth2Top();
            mScrolledY = 0;
        }
    }

    @Override
    protected boolean overScrollBy(int deltaX, int deltaY, int scrollX, int scrollY, int scrollRangeX, int scrollRangeY, int maxOverScrollX, int maxOverScrollY, boolean isTouchEvent) {
        log("overScrollBy: deltaX=" + deltaX + ", deltaY=" + deltaY + ", scrollX=" + scrollX + ", scrollRangeX=" + scrollRangeX);
        final boolean returnValue = super.overScrollBy(deltaX, deltaY, scrollX, scrollY, scrollRangeX,
                scrollRangeY, maxOverScrollX, maxOverScrollY, isTouchEvent);

        if (mLayout != null) {
            mScrolledY += deltaY;
            mLayout.scrollY(-deltaY);
        }
        return returnValue;
    }
{% endhighlight %}

## 小结
在处理手势的时候进场会遇到效果不一致，这时候通过适量的log，可以更好地帮助开发者理解问题的出处；
完整的代码可以参考在github上的地址,欢迎各种star，for，issue：

[https://github.com/avenwu/overscrolllayout](https://github.com/avenwu/overscrolllayout)


## 参考
* SwipeRefreshLayout
* Action-PullToRefresh