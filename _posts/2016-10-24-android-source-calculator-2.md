---
layout: post
title: "计算器：输入显示框的实现分析"
description: "Andorid源码分析"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/cover-andorid-source-calculator.png
keywords: "Andorid源码分析, 计算器"
tags: [Andorid源码分析]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/cover-andorid-source-calculator.png)

## 前言
在上一篇文章中，分析了计算器键盘面板的相关源码实现，本文将分析的是和键盘面板紧紧相连的输入显示框的实现；

## 别样的EditText
正如标题中提到的，通过分析我们可以知道，输入显示框用的是EditText，大致效果如下：

![device-2016-10-24-130734.mp4.gif]({{ site.baseurl }}/assets/images/calculator/device-2016-10-24-130734.mp4.gif)

注意观察其中的动画，细心的你可能已经发现两点特殊之处：

1. 输入框没有光标；
2. 点击输入框不会调起键盘，只能通过数字面板输入；
3. 显示的文字大小会随着长度而变化；

这几点不一样的地方，在本文将揭开他们的面纱。

## 自定义输入显示框
首先看看这个EditText，控件名为CalculatorEditText，约200行代码，里面主要的设置是针对文字大小和行高的处理.

## 光标去哪了
熟悉相关API的老司机应该知道设置光标是有现成接口，如果不知道也可以查一下EditText的API文档，在EditText中光标是cursor，经常可以设置blink的颜色，这里直接设置cursor不见，应该用的是setCursorVisible方法，但是在CalculatorEditText中搜索后并没有发现，相关设置，那么会不会是xml的style设置问题？
通过layout可知他的样式为DisplayEditTextStyle.Formula：

{% highlight xml%}
<style name="DisplayEditTextStyle.Formula">
    <item name="android:paddingTop">24dip</item>
    <item name="android:paddingBottom">8dip</item>
    <item name="android:paddingStart">16dip</item>
    <item name="android:paddingEnd">16dip</item>
    <item name="android:textSize">30sp</item>
</style>
{% endhighlight %}

也没有发现，继续找父类DisplayEditTextStyle：

{% highlight xml%}
<style name="DisplayEditTextStyle" parent="@android:style/Widget.Material.Light.EditText">
    <item name="android:background">@android:color/transparent</item>
    <item name="android:cursorVisible">false</item>
    <item name="android:fontFamily">sans-serif-light</item>
    <item name="android:includeFontPadding">false</item>
    <item name="android:gravity">bottom|end</item>
</style>
{% endhighlight %}
确实，cursor的设置作为基础样式被设置了android:cursorVisible为false

## 输入法的软键盘去哪了
现在在看看软键盘问题，就直觉来说，用了EditText应该会触发输入法软键盘才对，为什么计算器中没有出发呢？
其实这个是在CalculatorEditText中设置的，控件重载了onTouchEvent方法，直接取消了长按事件：

{% highlight java %}
@Override
public boolean onTouchEvent(MotionEvent event) {
    if (event.getActionMasked() == MotionEvent.ACTION_UP) {
        // Hack to prevent keyboard and insertion handle from showing.
        cancelLongPress();
    }
    return super.onTouchEvent(event);
}
{% endhighlight %}

从注释中也可以知道，就是这句 cancelLongPress 的调用阻止了键盘响应，而这个api点进去可以发现，是系统提供的接口。
如果把这段话全部注释掉那么，熟悉的键盘就回来了；

## 文字大小动起来
最后一点，输入文字的大小的动态变化。查看调用层，可以找到一个新增的监听器：

```
mFormulaEditText.setOnTextSizeChangeListener(this);
```
这个接口是自定义的，因此只需要分析接口的实现和定义逻辑就可以知道文字变化的原因。

监听器的定义，只有一个接口,将输入框自身和文字大小作为参数传递出来：

{% highlight java %}
public interface OnTextSizeChangeListener {
    void onTextSizeChanged(TextView textView, float oldSize);
}
{% endhighlight %}

搜索一下onTextSizeChanged的实现逻辑，可以找到对应实现逻辑，当输入框处于非输入状态时触发动画，动画为x,y轴的缩放和平移动画相结合，缩放因子由文字大小比决定，动画使用默认的AccelerateDecelerateInterpolator差值器，下面是具体代码：

{% highlight java %}
@Override
public void onTextSizeChanged(final TextView textView, float oldSize) {
    if (mCurrentState != CalculatorState.INPUT) {
        // Only animate text changes that occur from user input.
        return;
    }

    // Calculate the values needed to perform the scale and translation animations,
    // maintaining the same apparent baseline for the displayed text.
    final float textScale = oldSize / textView.getTextSize();
    final float translationX = (1.0f - textScale) *
            (textView.getWidth() / 2.0f - textView.getPaddingEnd());
    final float translationY = (1.0f - textScale) *
            (textView.getHeight() / 2.0f - textView.getPaddingBottom());

    final AnimatorSet animatorSet = new AnimatorSet();
    animatorSet.playTogether(
            ObjectAnimator.ofFloat(textView, View.SCALE_X, textScale, 1.0f),
            ObjectAnimator.ofFloat(textView, View.SCALE_Y, textScale, 1.0f),
            ObjectAnimator.ofFloat(textView, View.TRANSLATION_X, translationX, 0.0f),
            ObjectAnimator.ofFloat(textView, View.TRANSLATION_Y, translationY, 0.0f));
    animatorSet.setDuration(getResources().getInteger(android.R.integer.config_mediumAnimTime));
    animatorSet.setInterpolator(new AccelerateDecelerateInterpolator());
    animatorSet.start();
}
{% endhighlight %}

这里有个疑点，传入的oldSize和getTextSize的赋值时机，下面继续分析该接口的调用时机。
TextView有一个onTextChanged的方法，根据方法名和说明，该方法会在文字变化的时候被调用；从文章顶部的动画，可以知道，改变文字大小的会在文字长度变化的收触发

{% highlight java %}
@Override
protected void onTextChanged(CharSequence text, int start, int lengthBefore, int lengthAfter) {
    super.onTextChanged(text, start, lengthBefore, lengthAfter);

    final int textLength = text.length();
    if (getSelectionStart() != textLength || getSelectionEnd() != textLength) {
        // Pin the selection to the end of the current text.
        setSelection(textLength);
    }
    setTextSize(TypedValue.COMPLEX_UNIT_PX, getVariableTextSize(text.toString()));
}

@Override
public void setTextSize(int unit, float size) {
    final float oldTextSize = getTextSize();
    super.setTextSize(unit, size);

    if (mOnTextSizeChangeListener != null && getTextSize() != oldTextSize) {
        mOnTextSizeChangeListener.onTextSizeChanged(this, oldTextSize);
    }
}
{% endhighlight %}

重载的setTextSize在设置文字大小时，可以看到先前的疑问，及oldTextSize是在赋值前获取的；下面进一步看看这个这个对size的计算是如何处理的：

{% highlight java %}
public float getVariableTextSize(String text) {
    if (mWidthConstraint < 0 || mMaximumTextSize <= mMinimumTextSize) {
        // Not measured, bail early.
        return getTextSize();
    }

    // Capture current paint state.
    mTempPaint.set(getPaint());

    // Step through increasing text sizes until the text would no longer fit.
    float lastFitTextSize = mMinimumTextSize;
    while (lastFitTextSize < mMaximumTextSize) {
        final float nextSize = Math.min(lastFitTextSize + mStepTextSize, mMaximumTextSize);
        mTempPaint.setTextSize(nextSize);
        if (mTempPaint.measureText(text) > mWidthConstraint) {
            break;
        } else {
            lastFitTextSize = nextSize;
        }
    }

    return lastFitTextSize;
}
{% endhighlight %}
这里有几个比较关键的数字，分别是

* mWidthConstraint，控件所能显示的宽度值，相当于一个约束值
* mMaximumTextSize，设置的最大文字尺寸
* mMinimumTextSize，设置的最小文字尺寸
* mStepTextSize，文字增大的步长，数值上等于最大值减去最小值后差值取三分之一

了解了这几个变量的含义后可以分析一下getVariableTextSize的实现逻辑，首先根据这几个变量的取值合法性做判断，不满足则直接返回当前大小；
接着文字的大小从最小值开始和最大值比较，每次加上自增的步长，然后基于画笔测绘一下当前文字大小下，预期的大小是否超出我们的宽度约束；
大致就是这样一个循环比较逻辑。

## 小结
本文主要分析的输入框的一些小技术点，计算规则这一块后续单独分析；