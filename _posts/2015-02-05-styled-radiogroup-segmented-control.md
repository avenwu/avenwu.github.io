---
layout: post
title: "RadioGroup仿iOS Segmented Control样式"
header_image: /assets/img/2016-03-06-27.jpg
description: "自定义样式仿Segmented Control"
category: 
tags: [view,android]
---
{% include JB/setup %}
![img](/assets/img/2016-03-06-27.jpg)

## 前言
	
	代码获取
	git clone https://github.com/avenwu/support.git 


iOS中有一个Segmented Control组件，android中的RadioGroup与之类似，但是RadioGroup的默认样式不是很美观，但是只需要稍微调一下就可以长得和Segmented Control控件一样简洁优雅。

![ios_segmented_control.png](/assets/ios_segmented_control.png)

![radiogroup_button.png](/assets/radiogroup_button.png)

![radiogroup_button.png](/assets/styled_radiogroup.png)

## 实现
直接写style文件当然是最快的,只需设置每个RadioButton的对其为居中，修改默认的android:button资源，然后加上背景、文字的selector。

{% highlight xml %}
<RadioGroup
    android:layout_width="match_parent"
    android:layout_height="48dp"
    android:orientation="horizontal"
    android:gravity="center_vertical">

    <RadioButton
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="A"
        android:id="@+id/radioButton"
        style="@style/FlatRadioButtonStyle"
        android:background="@drawable/flat_round_shape_left" />

    <RadioButton
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="B"
        android:id="@+id/radioButton2"
        android:checked="true"
        style="@style/FlatRadioButtonStyle"
        android:background="@drawable/flat_round_shape_middle" />

    <RadioButton
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="C"
        android:id="@+id/radioButton3"
        style="@style/FlatRadioButtonStyle"
        android:background="@drawable/flat_round_shape_right" />
</RadioGroup>
{% endhighlight %}

style样式

{% highlight xml %}
<style name="FlatRadioButtonStyle">
    <item name="android:layout_weight">1</item>
    <item name="android:button">@null</item>
    <item name="android:gravity">center</item>
    <item name="android:textAppearance">?android:textAppearanceMedium</item>
    <item name="android:textColor">@color/radio_button_color</item>
    <item name="android:padding">5dp</item>
</style>
{% endhighlight %}

背景selector

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:state_checked="true">
        <shape android:shape="rectangle">
            <corners android:topLeftRadius="5dp" android:bottomLeftRadius="5dp" />
            <solid android:color="@android:color/white" />
            <stroke android:color="@android:color/white" android:width="1dp" />
        </shape>
    </item>
    <item>
        <shape android:shape="rectangle">
            <corners android:topLeftRadius="5dp" android:bottomLeftRadius="5dp" />
            <solid android:color="@android:color/transparent" />
            <stroke android:color="@android:color/white" android:width="1dp" />
        </shape>
    </item>

</selector>
{% endhighlight %}

这些都没什么问题，但是比较零散，每次都需要写很多的xml及其样式，selector等，所以可以做一些简单的封装，暴露一些必要的属性用于自定义，比如边框线的宽度，背景色等。

## 简单封装

自定义RadioGroup，将必要的初始化配置在内部完成。

{% highlight java %}
package net.avenwu.support.widget;

import android.content.Context;
import android.content.res.ColorStateList;
import android.content.res.TypedArray;
import android.graphics.Color;
import android.graphics.drawable.Drawable;
import android.graphics.drawable.GradientDrawable;
import android.graphics.drawable.StateListDrawable;
import android.util.AttributeSet;
import android.util.TypedValue;
import android.view.Gravity;
import android.view.View;
import android.view.ViewGroup;
import android.widget.RadioButton;
import android.widget.RadioGroup;

import net.avenwu.support.R;

/**
 * Created by chaobin on 2/4/15.
 */
public class FlatTabGroup extends RadioGroup {
    public FlatTabGroup(Context context) {
        this(context, null);
    }

    private int mRadius;
    private int mStroke;
    private int mHighlightColor;
    private String[] mItemString;
    private float mTextSize;
    private ColorStateList mTextColor;

    public FlatTabGroup(Context context, AttributeSet attrs) {
        super(context, attrs);
        setOrientation(HORIZONTAL);
        setGravity(Gravity.CENTER_VERTICAL);
        TypedArray array = context.obtainStyledAttributes(attrs, R.styleable.FlatTabGroup);
        mHighlightColor = array.getColor(R.styleable.FlatTabGroup_tab_border_color, Color.WHITE);
        mStroke = array.getDimensionPixelSize(R.styleable.FlatTabGroup_tab_border_width, 2);
        mRadius = array.getDimensionPixelOffset(R.styleable.FlatTabGroup_tab_radius, 5);
        mTextColor = array.getColorStateList(R.styleable.FlatTabGroup_tab_textColor);
        mTextSize = array.getDimensionPixelSize(R.styleable.FlatTabGroup_tab_textSize, 14);
        array.recycle();
        int id = array.getResourceId(R.styleable.FlatTabGroup_tab_items, 0);
        mItemString = isInEditMode() ? new String[]{"TAB A", "TAB B", "TAB C"} : context.getResources().getStringArray(id);
        generateTabView(context, attrs);
        updateChildBackground();
    }

    private void generateTabView(Context context, AttributeSet attrs) {
        if (mItemString == null) {
            return;
        }
        for (String text : mItemString) {
            RadioButton button = new RadioButton(context, attrs);
            button.setGravity(Gravity.CENTER);
            button.setButtonDrawable(android.R.color.transparent);
            button.setText(text);
            button.setTextColor(mTextColor);
            button.setTextSize(TypedValue.COMPLEX_UNIT_PX, mTextSize);
            addView(button, new LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT, 1));
        }
    }

    @Override
    protected void onFinishInflate() {
        super.onFinishInflate();
        updateChildBackground();
    }

    private void updateChildBackground() {
        for (int i = 0; i < getChildCount(); i++) {
            View child = getChildAt(i);
            if (child instanceof RadioButton) {
                child.setBackgroundDrawable(generateTabBackground(i, mHighlightColor));
            }
        }
    }

    private Drawable generateTabBackground(int position, int color) {
        StateListDrawable stateListDrawable = new StateListDrawable();
        stateListDrawable.addState(new int[]{android.R.attr.state_checked}, generateDrawable(position, color));
        stateListDrawable.addState(new int[]{}, generateDrawable(position, Color.TRANSPARENT));
        return stateListDrawable;
    }

    private Drawable generateDrawable(int position, int color) {
        float[] radius;
        if (position == 0) {
            radius = new float[]{
                    mRadius, mRadius,
                    0, 0,
                    0, 0,
                    mRadius, mRadius
            };
        } else if (position == getChildCount() - 1) {
            radius = new float[]{
                    0, 0,
                    mRadius, mRadius,
                    mRadius, mRadius,
                    0, 0
            };
        } else {
            radius = new float[]{
                    0, 0,
                    0, 0,
                    0, 0,
                    0, 0
            };
        }
        GradientDrawable shape = new GradientDrawable();
        shape.setCornerRadii(radius);
        shape.setColor(color);
        shape.setStroke(mStroke, mHighlightColor);
        return shape;
    }

}
{% endhighlight %}

**属性**

定义需要暴露给外面的属性

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <declare-styleable name="FlatTabGroup">
        <attr name="tab_items" format="reference" />
        <attr name="tab_border_width" format="dimension|reference" />
        <attr name="tab_border_color" format="color|reference" />
        <attr name="tab_radius" format="dimension|reference" />
        <attr name="tab_textColor" format="dimension|reference" />
        <attr name="tab_textSize" format="dimension|reference" />
    </declare-styleable>
</resources>
{% endhighlight %}

现在需要写一个RadioGroup时只需要少量的的配置：

- **app:tab_border_width="1dp"** 边线宽度
- **app:tab_border_color="@android:color/white"** 边线颜色
- **app:tab_items="@array/demo_array"** tabs字符串数组
- **app:tab_radius="5dp"** 边框弧度
- **app:tab_textSize="16sp"** 文本字号
- **app:tab_textColor="@color/radio_button_color_light_blue"** 文本颜色selector

示例：

{% highlight xml %}
<net.avenwu.support.widget.FlatTabGroup
                android:layout_width="match_parent"
                android:layout_height="40dp"
                app:tab_border_width="1dp"
                app:tab_border_color="@android:color/white"
                app:tab_items="@array/demo_array"
                app:tab_radius="5dp"
                app:tab_textSize="16sp"
                app:tab_textColor="@color/radio_button_color_light_blue"
                android:paddingTop="5dp"
                android:paddingBottom="5dp" />
{% endhighlight %}

所以现在写RadioGroup就非常方便，只需要根据需求配置相应属性即可。比如实现文章开头的效果，可以这样：

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        android:orientation="vertical"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:id="@+id/ll_container">

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:padding="20dp"
            android:orientation="vertical"
            android:background="@android:color/holo_red_dark">

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="RadioGroup with custom style"
                android:textColor="@android:color/white"
                android:layout_marginTop="10dp"
                android:layout_marginBottom="10dp" />

            <RadioGroup
                android:layout_width="match_parent"
                android:layout_height="48dp"
                android:orientation="horizontal"
                android:gravity="center_vertical">

                <RadioButton
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:text="A"
                    android:id="@+id/radioButton"
                    style="@style/FlatRadioButtonStyle"
                    android:background="@drawable/flat_round_shape_left" />

                <RadioButton
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:text="B"
                    android:id="@+id/radioButton2"
                    android:checked="true"
                    style="@style/FlatRadioButtonStyle"
                    android:background="@drawable/flat_round_shape_middle" />

                <RadioButton
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:text="C"
                    android:id="@+id/radioButton3"
                    style="@style/FlatRadioButtonStyle"
                    android:background="@drawable/flat_round_shape_right" />
            </RadioGroup>

            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="Custom FlatTabGroup extend RadioGroup"
                android:textColor="@android:color/white"
                android:layout_marginTop="10dp" />

        </LinearLayout>

        <FrameLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:padding="20dp"
            android:background="@android:color/holo_red_dark">

            <net.avenwu.support.widget.FlatTabGroup
                android:layout_width="match_parent"
                android:layout_height="40dp"
                app:tab_border_width="1dp"
                app:tab_border_color="@android:color/white"
                app:tab_items="@array/demo_array"
                app:tab_radius="5dp"
                app:tab_textSize="16sp"
                app:tab_textColor="@color/radio_button_color"
                android:paddingTop="5dp"
                android:paddingBottom="5dp" />

        </FrameLayout>

        <FrameLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:padding="20dp"
            android:background="@color/tab_orange">

            <net.avenwu.support.widget.FlatTabGroup
                android:layout_width="match_parent"
                android:layout_height="40dp"
                app:tab_border_width="1dp"
                app:tab_border_color="@android:color/white"
                app:tab_items="@array/demo_array"
                app:tab_radius="5dp"
                app:tab_textSize="16sp"
                app:tab_textColor="@color/radio_button_color_orange"
                android:paddingTop="5dp"
                android:paddingBottom="5dp" />

        </FrameLayout>

        <FrameLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:padding="20dp"
            android:background="@color/tab_purple">

            <net.avenwu.support.widget.FlatTabGroup
                android:layout_width="match_parent"
                android:layout_height="40dp"
                app:tab_border_width="1dp"
                app:tab_border_color="@android:color/white"
                app:tab_items="@array/demo_array"
                app:tab_radius="5dp"
                app:tab_textSize="16sp"
                app:tab_textColor="@color/radio_button_color_purple"
                android:paddingTop="5dp"
                android:paddingBottom="5dp" />

        </FrameLayout>

        <FrameLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:padding="20dp"
            android:background="@color/tab_blue">

            <net.avenwu.support.widget.FlatTabGroup
                android:layout_width="match_parent"
                android:layout_height="40dp"
                app:tab_border_width="1dp"
                app:tab_border_color="@android:color/white"
                app:tab_items="@array/demo_array"
                app:tab_radius="5dp"
                app:tab_textSize="16sp"
                app:tab_textColor="@color/radio_button_color_blue"
                android:paddingTop="5dp"
                android:paddingBottom="5dp" />

        </FrameLayout>

        <FrameLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:padding="20dp"
            android:background="@color/tab_light_blue">

            <net.avenwu.support.widget.FlatTabGroup
                android:layout_width="match_parent"
                android:layout_height="40dp"
                app:tab_border_width="1dp"
                app:tab_border_color="@android:color/white"
                app:tab_items="@array/demo_array"
                app:tab_radius="5dp"
                app:tab_textSize="16sp"
                app:tab_textColor="@color/radio_button_color_light_blue"
                android:paddingTop="5dp"
                android:paddingBottom="5dp" />

        </FrameLayout>

        <FrameLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:padding="20dp"
            android:background="@color/tab_green">

            <net.avenwu.support.widget.FlatTabGroup
                android:layout_width="match_parent"
                android:layout_height="40dp"
                app:tab_border_width="1dp"
                app:tab_border_color="@android:color/white"
                app:tab_items="@array/demo_array"
                app:tab_radius="5dp"
                app:tab_textSize="16sp"
                app:tab_textColor="@color/radio_button_color_green"
                android:paddingTop="5dp"
                android:paddingBottom="5dp" />

        </FrameLayout>

    </LinearLayout>
</ScrollView>
{% endhighlight %}

颜色selector

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:color="@color/tab_light_blue" android:state_checked="true" />
    <item android:color="@android:color/white" />
</selector>
{% endhighlight %}

tab数组

{% highlight xml %}
<string-array name="demo_array">
    <item>A</item>
    <item>B</item>
    <item>C</item>
</string-array>
{% endhighlight %}

## 小结
完整代码可以再这里获取：

[https://github.com/avenwu/support/blob/master/sample/src/main/java/com/avenwu/deepinandroid/StyledRadioButtonDemo.java](https://github.com/avenwu/support/blob/master/sample/src/main/java/com/avenwu/deepinandroid/StyledRadioButtonDemo.java)
