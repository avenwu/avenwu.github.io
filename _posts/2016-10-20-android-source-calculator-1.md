---
layout: post
title: "计算器：键盘面板的实现分析"
description: "Andorid源码分析"
header_image: /assets/img/cover-andorid-source-calculator.png
keywords: "Andorid源码分析, 计算器"
tags: [源码分析]
---
{% include JB/setup %}
![img](/assets/img/cover-andorid-source-calculator.png)

## 前言
Andorid5.+ 之后，系统自带的程式用户体验都很不错，本文作为分析计算器实现的第一篇文章，从输入的数字面板开始扒一下大厂的app设计；

![计算器]({{ site.baseurl }}/assets/images/calculator/计算器.png)


## 逼格从自定义数字面板开始
打开计算器，看到的界面类似上图，上半部分是输入/结果显示区域，下半部分是“九宫格”的数字和操作运算符；这个九宫格站在开发者的立场，可能会使用诸如GridView，嵌套的LinearLayout等等实现方案；

但是，大厂的答案是，都不；直接定义了一个轻量级的九宫格容器CalculatorPadLayout，可以用于数字，操作符，科学运算操作符这三大块；
下面来看看这个九宫格的实现；

首先定义了两个大家都熟悉的属性，用于设置行，列数；

```
private int mRowCount;
private int mColumnCount;
```

整个布局继承自ViewGroup,使用MarginLayoutParams作为布局控制；
最关键的实际上就是onLayout，在这里组行放置元素；和GridView布局类似，这里是等分的，通过前面设置的列数，计算每个元素的宽度，和高度

{% highlight java %}
@Override
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    final int paddingLeft = getPaddingLeft();
    final int paddingRight = getPaddingRight();
    final int paddingTop = getPaddingTop();
    final int paddingBottom = getPaddingBottom();

    final boolean isRTL = getLayoutDirection() == LAYOUT_DIRECTION_RTL;
    //计算单个元素的宽度，等分
    final int columnWidth =
            Math.round((float) (right - left - paddingLeft - paddingRight)) / mColumnCount;
    //计算单个元素的高度，等分
    final int rowHeight =
            Math.round((float) (bottom - top - paddingTop - paddingBottom)) / mRowCount;

    int rowIndex = 0, columnIndex = 0;
    for (int childIndex = 0; childIndex < getChildCount(); ++childIndex) {
        final View childView = getChildAt(childIndex);
        if (childView.getVisibility() == View.GONE) {
            continue;
        }

        final MarginLayoutParams lp = (MarginLayoutParams) childView.getLayoutParams();

        final int childTop = paddingTop + lp.topMargin + rowIndex * rowHeight;
        final int childBottom = childTop - lp.topMargin - lp.bottomMargin + rowHeight;
        final int childLeft = paddingLeft + lp.leftMargin +
                (isRTL ? (mColumnCount - 1) - columnIndex : columnIndex) * columnWidth;
        final int childRight = childLeft - lp.leftMargin - lp.rightMargin + columnWidth;

        final int childWidth = childRight - childLeft;
        final int childHeight = childBottom - childTop;
        if (childWidth != childView.getMeasuredWidth() ||
                childHeight != childView.getMeasuredHeight()) {
            childView.measure(
                    MeasureSpec.makeMeasureSpec(childWidth, MeasureSpec.EXACTLY),
                    MeasureSpec.makeMeasureSpec(childHeight, MeasureSpec.EXACTLY));
        }
        childView.layout(childLeft, childTop, childRight, childBottom);
        //更新行列索引
        rowIndex = (rowIndex + (columnIndex + 1) / mColumnCount) % mRowCount;
        columnIndex = (columnIndex + 1) % mColumnCount;
    }
}
{% endhighlight %}

## 键盘数字国际化
上面的九宫格基本可以直接用来作为数字键盘放置数字，这里需要介绍对应的数字键盘的实现类CalculatorNumericPadLayout，该类继承自CalculatorPadLayout，区别在于对数字的展示处理。

第一眼看到这个类的实现时觉得挺奇怪的，为什么数字需要动态的在inflate后设置，而具体的按钮却是在xml中定义；
其实这个和国际化有关，全世界绝大多数国家都是用阿拉伯数字在计数，但是也有例外，比如拉丁文，因此这里会根据Locale信息和配置，决定以哪种数字显示；

![latin numbers]({{ site.baseurl }}/assets/images/calculator/latin-numbers-1-10.png)

国际化有很多写法，没有用常规的values直接定义，猜测可能是出于计算显示的可配性，毕竟不是所有使用拉丁问的都不使用阿拉伯数字；
无论如何，我们还是先看看google是怎样设置的数字键盘：

首先在xml中配置了12个按钮，分别用于显示数字0-9，小数点，以及等号；

{% highlight xml %}
<com.android.calculator2.CalculatorNumericPadLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/pad_numeric"
    style="@style/PadLayoutStyle.Numeric"
    android:background="@color/pad_numeric_background_color"
    android:columnCount="3"
    android:rowCount="4">

    <Button
        android:id="@+id/digit_7"
        style="@style/PadButtonStyle.Numeric"
        android:onClick="onButtonClick" />

    <Button
        android:id="@+id/digit_8"
        style="@style/PadButtonStyle.Numeric"
        android:onClick="onButtonClick" />

    <Button
        android:id="@+id/digit_9"
        style="@style/PadButtonStyle.Numeric"
        android:onClick="onButtonClick" />

    <Button
        android:id="@+id/digit_4"
        style="@style/PadButtonStyle.Numeric"
        android:onClick="onButtonClick" />

    <Button
        android:id="@+id/digit_5"
        style="@style/PadButtonStyle.Numeric"
        android:onClick="onButtonClick" />

    <Button
        android:id="@+id/digit_6"
        style="@style/PadButtonStyle.Numeric"
        android:onClick="onButtonClick" />

    <Button
        android:id="@+id/digit_1"
        style="@style/PadButtonStyle.Numeric"
        android:onClick="onButtonClick" />

    <Button
        android:id="@+id/digit_2"
        style="@style/PadButtonStyle.Numeric"
        android:onClick="onButtonClick" />

    <Button
        android:id="@+id/digit_3"
        style="@style/PadButtonStyle.Numeric"
        android:onClick="onButtonClick" />

    <Button
        android:id="@+id/dec_point"
        style="@style/PadButtonStyle.Numeric"
        android:onClick="onButtonClick" />

    <Button
        android:id="@+id/digit_0"
        style="@style/PadButtonStyle.Numeric"
        android:onClick="onButtonClick" />

    <Button
        android:id="@+id/eq"
        style="@style/PadButtonStyle.Numeric.Equals"
        android:contentDescription="@string/desc_eq"
        android:onClick="onButtonClick"
        android:text="@string/eq" />

</com.android.calculator2.CalculatorNumericPadLayout>
{% endhighlight %}

这些按钮都设置了相同的点击事件，数字键都没有设置文案，有动态配置，每个按钮有独立的id，用于动态识别；

设置数字的代码在layout加载完毕后的onFinishInflate方法中，如下：

{% highlight java %}
@Override
public void onFinishInflate() {
    super.onFinishInflate();

    Locale locale = getResources().getConfiguration().locale;
    if (!getResources().getBoolean(R.bool.use_localized_digits)) {
        locale = new Locale.Builder()
            .setLocale(locale)
            .setUnicodeLocaleKeyword("nu", "latn")
            .build();
    }

    final DecimalFormatSymbols symbols = DecimalFormatSymbols.getInstance(locale);
    final char zeroDigit = symbols.getZeroDigit();
    for (int childIndex = getChildCount() - 1; childIndex >= 0; --childIndex) {
        final View v = getChildAt(childIndex);
        if (v instanceof Button) {
            final Button b = (Button) v;
            switch (b.getId()) {
                case R.id.digit_0:
                    b.setText(String.valueOf(zeroDigit));
                    break;
                case R.id.digit_1:
                    b.setText(String.valueOf((char) (zeroDigit + 1)));
                    break;
                case R.id.digit_2:
                    b.setText(String.valueOf((char) (zeroDigit + 2)));
                    break;
                case R.id.digit_3:
                    b.setText(String.valueOf((char) (zeroDigit + 3)));
                    break;
                case R.id.digit_4:
                    b.setText(String.valueOf((char) (zeroDigit + 4)));
                    break;
                case R.id.digit_5:
                    b.setText(String.valueOf((char) (zeroDigit + 5)));
                    break;
                case R.id.digit_6:
                    b.setText(String.valueOf((char) (zeroDigit + 6)));
                    break;
                case R.id.digit_7:
                    b.setText(String.valueOf((char) (zeroDigit + 7)));
                    break;
                case R.id.digit_8:
                    b.setText(String.valueOf((char) (zeroDigit + 8)));
                    break;
                case R.id.digit_9:
                    b.setText(String.valueOf((char) (zeroDigit + 9)));
                    break;
                case R.id.dec_point:
                    b.setText(String.valueOf(symbols.getDecimalSeparator()));
                    break;
            }
        }
    }
}
{% endhighlight %}

## 运算符
这里实际上就是加减乘除的运算符；和数字面板基本一致，只不过不需要动态设置文案，因此可以直接用CalculatorPadLayout作为容器

{% highlight xml %}
<com.android.calculator2.CalculatorPadLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/pad_operator"
    style="@style/PadLayoutStyle.Operator"
    android:background="@color/pad_operator_background_color"
    android:columnCount="1"
    android:rowCount="5">

    <Button
        android:id="@+id/del"
        style="@style/PadButtonStyle.Operator.Text"
        android:contentDescription="@string/desc_del"
        android:onClick="onButtonClick"
        android:text="@string/del" />

    <Button
        android:id="@+id/clr"
        style="@style/PadButtonStyle.Operator.Text"
        android:contentDescription="@string/desc_clr"
        android:onClick="onButtonClick"
        android:text="@string/clr"
        android:visibility="gone" />

    <Button
        android:id="@+id/op_div"
        style="@style/PadButtonStyle.Operator"
        android:contentDescription="@string/desc_op_div"
        android:onClick="onButtonClick"
        android:text="@string/op_div" />

    <Button
        android:id="@+id/op_mul"
        style="@style/PadButtonStyle.Operator"
        android:contentDescription="@string/desc_op_mul"
        android:onClick="onButtonClick"
        android:text="@string/op_mul" />

    <Button
        android:id="@+id/op_sub"
        style="@style/PadButtonStyle.Operator"
        android:contentDescription="@string/desc_op_sub"
        android:onClick="onButtonClick"
        android:text="@string/op_sub" />

    <Button
        android:id="@+id/op_add"
        style="@style/PadButtonStyle.Operator"
        android:contentDescription="@string/desc_op_add"
        android:onClick="onButtonClick"
        android:text="@string/op_add" />

</com.android.calculator2.CalculatorPadLayout>
{% endhighlight %}

## 科学计算运算符
除了执行简单的加减乘除，计算器一般都会支持一些常用的科学基础，比如求根，三角运算等；
{% highlight xml %}
<com.android.calculator2.CalculatorPadLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/pad_advanced"
    style="@style/PadLayoutStyle.Advanced"
    android:background="@color/pad_advanced_background_color">

    <Button
        android:id="@+id/fun_sin"
        style="@style/PadButtonStyle.Advanced"
        android:contentDescription="@string/desc_fun_sin"
        android:onClick="onButtonClick"
        android:text="@string/fun_sin" />

    <Button
        android:id="@+id/fun_cos"
        style="@style/PadButtonStyle.Advanced"
        android:contentDescription="@string/desc_fun_cos"
        android:onClick="onButtonClick"
        android:text="@string/fun_cos" />

    <Button
        android:id="@+id/fun_tan"
        style="@style/PadButtonStyle.Advanced"
        android:contentDescription="@string/desc_fun_tan"
        android:onClick="onButtonClick"
        android:text="@string/fun_tan" />

    <Button
        android:id="@+id/fun_ln"
        style="@style/PadButtonStyle.Advanced"
        android:contentDescription="@string/desc_fun_ln"
        android:onClick="onButtonClick"
        android:text="@string/fun_ln" />

    <Button
        android:id="@+id/fun_log"
        style="@style/PadButtonStyle.Advanced"
        android:contentDescription="@string/desc_fun_log"
        android:onClick="onButtonClick"
        android:text="@string/fun_log" />

    <Button
        android:id="@+id/op_fact"
        style="@style/PadButtonStyle.Advanced"
        android:contentDescription="@string/desc_op_fact"
        android:onClick="onButtonClick"
        android:text="@string/op_fact" />

    <Button
        android:id="@+id/const_pi"
        style="@style/PadButtonStyle.Advanced"
        android:contentDescription="@string/desc_const_pi"
        android:onClick="onButtonClick"
        android:text="@string/const_pi" />

    <Button
        android:id="@+id/const_e"
        style="@style/PadButtonStyle.Advanced"
        android:contentDescription="@string/desc_const_e"
        android:onClick="onButtonClick"
        android:text="@string/const_e" />

    <Button
        android:id="@+id/op_pow"
        style="@style/PadButtonStyle.Advanced"
        android:contentDescription="@string/desc_op_pow"
        android:onClick="onButtonClick"
        android:text="@string/op_pow" />

    <Button
        android:id="@+id/lparen"
        style="@style/PadButtonStyle.Advanced"
        android:contentDescription="@string/desc_lparen"
        android:onClick="onButtonClick"
        android:text="@string/lparen" />

    <Button
        android:id="@+id/rparen"
        style="@style/PadButtonStyle.Advanced"
        android:contentDescription="@string/desc_rparen"
        android:onClick="onButtonClick"
        android:text="@string/rparen" />

    <Button
        android:id="@+id/op_sqrt"
        style="@style/PadButtonStyle.Advanced"
        android:contentDescription="@string/desc_op_sqrt"
        android:onClick="onButtonClick"
        android:text="@string/op_sqrt" />

</com.android.calculator2.CalculatorPadLayout>
{% endhighlight %}
## 竖屏模式
在竖屏下，操作键盘包括的上述三部分无法全部一起显示出来，因此这里是折叠的，实现上用了ViewPager显示为两屏，这里因为用的是Viewpager实现，因此还是比较有意思的，下面一起看一下这个分屏是如何实现的。
![输入面板-竖屏]({{ site.baseurl }}/assets/images/calculator/device-2016-10-20-180046.png)
{% highlight xml %}
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <include
        layout="@layout/display"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

    <com.android.calculator2.CalculatorPadViewPager
        android:id="@+id/pad_pager"
        android:layout_width="match_parent"
        android:layout_height="0dip"
        android:layout_weight="1"
        android:overScrollMode="never">

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="match_parent">

            <include layout="@layout/pad_numeric" />
            <include layout="@layout/pad_operator_one_col" />

        </LinearLayout>

        <include layout="@layout/pad_advanced" />

    </com.android.calculator2.CalculatorPadViewPager>

</LinearLayout>
{% endhighlight %}

CalculatorPadViewPager即是我们提到的ViewPager,内部设置了一些api来实现不同于常规Viewpager滑动的效果；
首先是PageTransformer，这个经常用在改变Viewpager每页的切换动画，因此也可以用来改变view的位移量；

{% highlight java %}
private final PageTransformer mPageTransformer = new PageTransformer() {
    @Override
    public void transformPage(View view, float position) {
        if (position < 0.0f) {
            // Pin the left page to the left side.
            view.setTranslationX(getWidth() * -position);
            view.setAlpha(Math.max(1.0f + position, 0.0f));
        } else {
            // Use the default slide transition when moving to the next page.
            view.setTranslationX(0.0f);
            view.setAlpha(1.0f);
        }
    }
};
{% endhighlight %}

另一个是OnPageChangeListener,当切屏后，递归设置键盘按钮的可用性，

{% highlight java %}
private final OnPageChangeListener mOnPageChangeListener = new SimpleOnPageChangeListener() {
    private void recursivelySetEnabled(View view, boolean enabled) {
        if (view instanceof ViewGroup) {
            final ViewGroup viewGroup = (ViewGroup) view;
            for (int childIndex = 0; childIndex < viewGroup.getChildCount(); ++childIndex) {
                recursivelySetEnabled(viewGroup.getChildAt(childIndex), enabled);
            }
        } else {
            view.setEnabled(enabled);
        }
    }

    @Override
    public void onPageSelected(int position) {
        if (getAdapter() == mStaticPagerAdapter) {
            for (int childIndex = 0; childIndex < getChildCount(); ++childIndex) {
                // Only enable subviews of the current page.
                recursivelySetEnabled(getChildAt(childIndex), childIndex == position);
            }
        }
    }
};
{% endhighlight %}

## 横屏模式
上面的分析基板上也适用于横屏模式，计算器默认是支持横竖屏的，在横屏下，键盘是全部展开的，而不向竖屏那样由于显示空间的问题折叠起来；
![输入面板-横屏]({{ site.baseurl }}/assets/images/calculator/device-2016-10-21-141204.png)
{% highlight xml%}
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <include
        layout="@layout/display"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="0dip"
        android:layout_weight="1">

        <include layout="@layout/pad_numeric" />
        <include layout="@layout/pad_operator_two_col" />
        <include layout="@layout/pad_advanced" />

    </LinearLayout>

</LinearLayout>
{% endhighlight %}

## 小结
在Andorid自带程序中，计算器相对来说工程量比较小，分析起来也比较容易，了解了每个环节的实现后，就可以根据个人喜好进行二次开发了，这一点做ROM的开发人员应该特别熟悉

分析用的代码院子google源码，也可以在这里获取：[计算器源码]({{ site.baseurl }}/assets/files/platform_packages_apps_calculator.tar.gz)


