---
layout: post
title: "【MIUI】Muisc App之音频播放动画"
description: ""
header_image: /assets/img/2017-07-23-01.png
keywords: "音频动画"
tags: [ROM]
---
{% include JB/setup %}
![img](/assets/img/2017-07-23-01.png)

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

为了确认问题，利用脚本匹配了ROM中所有的odex文件，不过还是没有找到线索；这样的话就比较尴尬了，要么MIUI有一套机制会在安装过程中动态合并ROM的预装app，但即使是这样，这个文件也应该会在ROM内出现，现在连关键字都匹配不出来，所以也有可能是版本问题；

# 消失的WaveVisualizer

首先我们是根据MIUI的系统版本下载的对应ROM，但是这里面也不代表这个ROM和手机系统就是完全一致的；带着这个疑问我们去检查一下。

![miui-version]({{ site.baseurl }}/assets/images/miui-version.png)

![IMG_20170725_112646]({{ site.baseurl }}/assets/images/IMG_20170725_112646.jpg)

这两张图我们可以确认MIUI的大版本是一致的，同时ROM这边还可以看到一个类似git摘要的信息；为了进一步确认是不是APK自身的差异，我们把手机中安装的播放器的apk提取出来，和ROM中的apk作比较；

![miui-player-2.9]({{ site.baseurl }}/assets/images/miui-player-2.9.png)

![miui-player-2.10]({{ site.baseurl }}/assets/images/miui-player-2.10.png)

原来如此，虽然ROM的大版本和手机系统一致，但是其内部的app却有差异，应该是预装的低版本，在使用过程中，手机上的app是可以升级版本的，因此在分析ROM的时候有可能代码是不一致，这取决于目标app的版本是否一致；

现在我们只需要要获取2.10版本的播放器，并分析其中是否包含WaveVisualizer；

```
aven-mac-pro-2:~ aven$ check Desktop/base.apk WaveVisualizer
Check target whether it conatins WaveVisualizer ...
Keyword: WaveVisualizer
APK: Desktop/base.apk
Checking ...
Warnning!!! We found 4 times of WaveVisualizer inside your Desktop/base.apk
Keyword found in target -_-
```

果然新的APK中包含WaveVisualizer。


# WaveVisualizer实现

现在我们看下他的绘制内容：

```
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    Path path = this.mTmpPath;
    long cur = SystemClock.elapsedRealtime();
    long timeDur = cur - this.mLastDrawTime;
    this.mLastDrawTime = cur;
    for (WaveLine b : this.mWaves) {
        b.draw(timeDur, canvas, this.mPaint, path);
        path.reset();
    }
    canvas.restore();
    if (this.mIsDoingAnimation) {
        invalidate();
    }
}
```

直觉告诉我这次找对了，这里循环绘制了一组WaveLine,每条WaveLine的绘制由其内部实现，通过绘制path得到类似的波形图，而path每次都是通过mTmpPath赋值，所以会有一个改变其值的过程；

经过一些列调用，最后绘制了Bezier曲线，方法如下，并且在改变坐标值是，还利用了android.media.audiofx.Visualizer.OnDataCaptureListener，看名字应该是音频播放过程中的一些频率回调之类的接口。

```
private void drawBezier(Point s, Point e, Path path, int bl) {
    if (s != null && e != null && path != null) {
        path.lineTo((float) s.x, (float) s.y);
        if (Math.abs(e.x - s.x) < bl * 2) {
            bl = Math.abs(e.x - s.x) / 2;
        }
        path.cubicTo((float) (s.x + bl), (float) s.y, (float) (e.x - bl), (float) e.y, (float) e.x, (float) e.y);
    }
}
```

要分析清楚整个波形的逻辑，基本逻辑都在WaveLine和WaveVisualizer之中。

# 小结

由于将重点放在了ROM，导致在提取APK后忽略了APK的版本差异，走了些弯路；
至此，从ROM中提取并分析app的整个流程就大功告成了，以后看到什么有意思的交互都可以类似的去分析源码；
