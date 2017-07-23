---
layout: post
title: "【MIUI】Muisc App之音频播放动画"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2017-07-23-01.png
keywords: "音频动画"
tags: [MIUI]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2017-07-23-01.png)

# 前言

现在终于到了解MIUI的音频动画的时刻了。之前看过一篇TOS 录音机动画的技术文章，不知道MIUI的音频动画又是如何实现的呢？

# 播放界面

我们知道播放音乐的主界面是MusicBrowserActivity，底部有一个TAB。

![MusicBrowserActivity]({{ site.baseurl }}/assets/images/device-2017-07-23-121832.gif)

通过分析，底部TAB上谱图对应的View比较可疑的有以下两个：

```
com.miui.player.phone.view.NowplayingBar{5a40c0 G.E...C.. ......I. 0,1763-1080,1920 #7f0e013b app:id/nowplaying_bar}
com.miui.player.view.SwitchImage{da7c3c6 V.ED..... ........ 37,29-136,128 #7f0e000b app:id/album}
com.miui.player.view.DanceBar{f38df87 G.ED..... ......I. 0,0-0,0 #7f0e0042 app:id/play_indicator}
android.widget.LinearLayout{8bc7eb4 V.E...... ........ 136,33-719,123 #7f0e002f app:id/title_wrapper}
  android.widget.TextView{a14fadd G.ED..... ......ID 0,0-0,0 #7f0e0159 app:id/hint_text}
  android.widget.TextView{34c5652 V.ED..... ........ 28,0-546,49 #7f0e0031 app:id/primary_text}
  android.widget.TextView{f39ed23 V.ED..... ........ 28,52-546,90 #7f0e0032 app:id/secondary_text}
com.miui.player.phone.view.PlayController{dfcd220 V.E...... ........ 719,25-1043,132 #7f0e00b0 app:id/play_control}
  android.widget.ImageView{16b67d9 V.ED..C.. ........ 55,0-162,107 #2}
  android.widget.ImageView{f35859e V.ED..C.. ........ 217,0-324,107}
```

```
com.miui.player.view.NowplayingCircle{825c3f6 V.E...... .......D 127,1062-952,1887}
  android.widget.ImageView{d93f7 V.ED..... ........ 327,654-498,825}
  android.widget.ImageView{e77264 V.ED..... ........ 327,654-498,825}
  android.widget.ImageView{4be5ccd V.ED..... ........ 327,654-498,825}
  android.widget.FrameLayout{8e22982 V.E...... .......D 348,696-477,825 #7f0e003f app:id/content}
    com.miui.player.view.WaveVisualizer{d7d7893 V.ED..... ......ID 0,0-129,129}

```
如此找到dex中的com.miui.player.view.DanceBar，WaveVisualizer即可分析相应逻辑，进一步确认；

# 波谱视图

反查APK后，我们可以的得到DanceBar，通过分析绘制部分，可疑知道DanceBar并不是波谱图：
 
{% highlight java %}
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    for (int i = 0; i < this.mBars.length; i += MESSAGE_INVALIDATE) {
        this.mPaint.setAlpha(HttpStatus.SC_OK);
        this.mRect.set((this.mRectWidth * i) + (this.mInteWidth * i), (this.mHeight - this.mBars[i].volume) + this.mDanceMinHeight, ((i + MESSAGE_INVALIDATE) * this.mRectWidth) + (this.mInteWidth * i), this.mHeight);
        canvas.drawRect(this.mRect, this.mPaint);
        this.mPaint.setAlpha(100);
        this.mRect.set((this.mRectWidth * i) + (this.mInteWidth * i), (this.mHeight - this.mBars[i].volume) + (this.mDanceMinHeight / 2), ((i + MESSAGE_INVALIDATE) * this.mRectWidth) + (this.mInteWidth * i), (this.mHeight - this.mBars[i].volume) + this.mDanceMinHeight);
        canvas.drawRect(this.mRect, this.mPaint);
        this.mPaint.setAlpha(25);
        this.mRect.set((this.mRectWidth * i) + (this.mInteWidth * i), this.mHeight - this.mBars[i].volume, ((i + MESSAGE_INVALIDATE) * this.mRectWidth) + (this.mInteWidth * i), this.mBars[i].volume + (this.mDanceMinHeight / 2));
        canvas.drawRect(this.mRect, this.mPaint);
    }
    if (this.mIsDancing) {
        doInvalidateInInterval();
    }
}
{% endhighlight %}

我们看到绘制的时候，只有一个循环操作，绘制的都是透明度不同的Rect, 但是波谱肯定是不可能仅通过矩形就绘制出来的。实际上这DanceBar是列表中正在播放的条目上展示的矩形块；

并且从代码来看这个小的动态视图是一个随机的矩形块组合；

{% highlight java %}
private void moveToNextState() {
    Bar[] arr$ = this.mBars;
    int len$ = arr$.length;
    for (int i$ = 0; i$ < len$; i$ += MESSAGE_INVALIDATE) {
        Bar bar = arr$[i$];
        bar.volume += bar.volSpeed;
        if (bar.volume <= bar.volGap) {
            bar.volume = bar.volGap;
            bar.volSpeed = (this.mRandom.nextInt(this.mSpeedMax - this.mSpeedMin) + MESSAGE_INVALIDATE) + this.mSpeedMin;
            bar.volGap = (this.mRandom.nextInt(Math.max(this.mGapMax - this.mDanceMinHeight, MESSAGE_INVALIDATE)) + this.mDanceMinHeight) + MESSAGE_INVALIDATE;
        } else if (bar.volume >= this.mDanceMaxHeight) {
            bar.volume = this.mDanceMaxHeight;
            bar.volSpeed = -((this.mRandom.nextInt(this.mSpeedMax - this.mSpeedMin) + MESSAGE_INVALIDATE) + this.mSpeedMin);
        }
    }
}
{% endhighlight %}

所以需要继续查找另一个视图，不过很遗憾，在dex中并没有找到WaveVisualizer，这一点比较奇怪，难道播放器的apk还有其他没有打包进来的代码？

并且，我们简单匹配了ROM中所有的odex文件，都没有找到线索；

//TODO 未完待续

# 小结

