---
layout: post
title: "Porter/Duff，图片加遮罩setColorFilter"
header_image: /assets/img/2016-03-06-29.jpg
description: "图片叠加模式分析"
category: 
tags: [android,图片]
---
{% include JB/setup %}
![img](/assets/img/2016-03-06-29.jpg)

## 前言

经常会遇到给图片加蒙层/遮罩的需求，比如，头像上面需要一个半透明的黑色啊什么的，解决这种需求并不难，实现方案也很多，最生硬的可以直接在图片上再放一个view设置背景为半透明，或者自己写一个带透明效果的ImageView，或者巧妙的利用Android ImageView提供的一些属性如setColorFilter。下面分别实现三种方案。

![colorfilter.png](/assets/colorfilter.png)

## 添加额外视图
ImageView的父级用FrameLayout或RelativeLayout

{% highlight xml %}
<FrameLayout
    android:layout_width="0dp"
    android:layout_height="wrap_content"
    android:layout_weight="1">

    <ImageView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:src="@drawable/ic_watch" />

    <View
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:background="#99000000" />
</FrameLayout>
{% endhighlight %}

## 自定义ImageView
在onDraw中额外在绘制一个半透明即可。  

{% highlight java %}
public class DimImageView extends ImageView {
    public static int DEFAULT_DIM = 0x99000000;
    int mDimColor;

    public DimImageView(Context context) {
        this(context, null);
    }

    public DimImageView(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public DimImageView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        TypedArray array = context.obtainStyledAttributes(attrs, R.styleable.DimImageView, defStyleAttr, 0);
        mDimColor = array.getColor(R.styleable.DimImageView_dim, DEFAULT_DIM);
        array.recycle();
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        canvas.drawColor(mDimColor);
    }
}
{% endhighlight%}

## 利用PorterDuff
由于ImageView支持PorterDuff，所以了解相关属性的话，可以直接利用setColorFilter；

{% highlight java %}
static final int MASK_HINT_COLOR = 0x99000000;
mImage.setColorFilter(MASK_HINT_COLOR, mode);
{% endhighlight%}

## 小结
以上三种方式均可实现蒙层效果，但是第一种是最不好的，由于会增加不必要的视图层级。而自定义的好处是相对扩展性强，可以有更多地自定义控件。当然最方便的还是直接使用setColorFilter。

## 参考
- [http://blog.danlew.net/2014/08/18/fast-android-asset-theming-with-colorfilter/](http://blog.danlew.net/2014/08/18/fast-android-asset-theming-with-colorfilter/)
- [http://ssp.impulsetrain.com/porterduff.html](http://ssp.impulsetrain.com/porterduff.html)
- [http://www.ibm.com/developerworks/java/library/j-mer0918/](http://www.ibm.com/developerworks/java/library/j-mer0918/)
