---
layout: post
title: "避免无效的多次点击事件"
header_image: /assets/img/2016-03-06-07.jpg
description: ""
category: 
tags: [android]
---
{% include JB/setup %}
![img](/assets/img/2016-03-06-07.jpg)

## 前言

	问题总是会有的，有时候选择忽略


记得几年前，曾遇到过恶意点击引发的问题，大意是有一个按钮，点击后会触发一个动作，比开启一个新的页面，这个时候如果故意高频的点击，非常可能出现开启两次同一个页面；

针对这种问题，一下子如果要解决，仿佛整个人都不好了，因为所有的点击事件都可能存在；今日正好看到一段有意思的代码，顺便谈谈这个问题和处理思路。

## 处理思路

	那么什么情况下需要处理？又怎么处理？

正如文章开头说的，笔者认为不是所有问题都需要解决，当然该解决的还是得解决；

一般情况下，点击事件发生后，页面发生变化，则按钮很自然无法被再次点击，问题就出在，如果页面没有发生变化，或者操作频率足够高，肯定是会导致点击被多次处理的，一般来说需要通过中间变量来控制是否需要再次响应单击事件；

比如,假定300毫秒内只响应一次：

{% highlight java %}
long mLastTimestamp = 0;
final int INTERVAL = 300;

public void onClick(View view) {
	if(System.currentMillis() - mLastTimestamp >= INTERVAL) {
		// TODO do whatever you like
		mLastTimestamp = System.currentMills();
	}
}
{% endhighlight %}

如此，做一个适当的法制控制, 如果需要大量使用，可以稍作封装；

今日在耙代码的时候，偶然看见另一个变种，思路一样，但利用了系统的事件分发；

{% highlight java %}
/**
 * A {@linkplain View.OnClickListener click listener} that debounces multiple clicks posted in the
 * same frame. A click on one button disables all buttons for that frame.
 */
public abstract class DebouncingOnClickListener implements View.OnClickListener {
  static boolean enabled = true;

  private static final Runnable ENABLE_AGAIN = new Runnable() {
    @Override public void run() {
      enabled = true;
    }
  };

  @Override public final void onClick(View v) {
    if (enabled) {
      enabled = false;
      v.post(ENABLE_AGAIN);
      doClick(v);
    }
  }

  public abstract void doClick(View v);
}

{% endhighlight %}

可以看到，代码中通过View.post()在UI队列里提交了了一个更改控制开关的Runnable, 也就是必须等这个修改的事件被触发了，下次点击才能进入判断语句内部；

{% highlight java %}
   /**
     * <p>Causes the Runnable to be added to the message queue.
     * The runnable will be run on the user interface thread.</p>
     *
     * @param action The Runnable that will be executed.
     *
     * @return Returns true if the Runnable was successfully placed in to the
     *         message queue.  Returns false on failure, usually because the
     *         looper processing the message queue is exiting.
     *
     * @see #postDelayed
     * @see #removeCallbacks
     */
    public boolean post(Runnable action) {
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null) {
            return attachInfo.mHandler.post(action);
        }
        // Assume that post will succeed later
        ViewRootImpl.getRunQueue().post(action);
        return true;
    }
{% endhighlight %}

根据post的实现代码，我们知道，这个修改开关的润那边和ui操作事件分发实在同一个UI绘制线程被处理了，因此实现了屏蔽快速点击的效果；





