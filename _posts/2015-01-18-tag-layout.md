---
layout: post
title: "流式标签生成控件"
header_image: /assets/img/2016-03-06-32.jpg
description: ""
category: "customlayout"
tags: [view,android]
---
{% include JB/setup %}  
![img](/assets/img/2016-03-06-32.jpg)
## 前言
无图无真相，[完整代码](https://github.com/avenwu/support/blob/master/support/src/main/java/net/avenwu/support/widget/TagFlowLayout.java)  

![demo效果图](/assets/tag_input_layout_demo.gif)

## 思路

	* A 基于EditView，Html/Span样式变换
	* B 基于ViewGroup，自定义Layout
	
### Span样式
采用span需要解决如下难点：

* 同一EditView内样式混排
* 向前删除这个tag标签

要求对html，span相关属性了如指掌，需要计算每个标签的位置，删除判断，复杂度很高。

### 自定义Layout
自定义天生的优点就是任性，do what ever you like。  

* 定义标签view
* 动态添加，删除
* 样式定义配置

技术难点在于如何设计，怎么自定义相关标签，好处绕过删除的难点，复杂度低，偏技术性。

## 实现方案

暂且选择方案B，现在思考一下怎么设计整个交互实现。

### 设计思路

* 标签项，每一个即是一个Label，包括选中状态，可用TextView
* 输入标签，输入还是用EditView，需要和整体融合在一起
* 增加、删除标签相当于在容器内addView，removeChild
* 监听输入变化，确定分割符，如回车（Enter），逗号（Comma）触发标签生成逻辑，回退键（Delete/Back key）触发删除标签逻辑
* 细节处理，点击删除位置变化，选中高光，键盘显隐，自动换行等等
* 善后，诸如标签配置化问题

### 代码里看细节
此处省略2万字。。。。。  
现在我看几个关键性代码。另外相信认真看这篇文章的，绝大多数都是有经验的工程师，很容易理解。  

### 生成标签
第一波代码是标签项的生成, 每次实例化一个TextView，根据我们的需要设置相关属性，显示内容从EditView获取后清空输入控件，最后将标签view添加到布局中。

{% highlight java %}
private void generateTag(CharSequence tag) {
    CharSequence tagString = tag == null ? mInputView.getText().toString() : tag;
    mInputView.getText().clear();
    final int targetIndex = indexOfChild(mInputView);
    TextView tagLabel;
    if (mDecorator.getLayout() != INVALID_VALUE) {
        View view = View.inflate(getContext(), mDecorator.getLayout(), null);
        if (view instanceof TextView) {
            tagLabel = (TextView) view;
        } else {
            throw new IllegalArgumentException("The custom layout for tagLabel label must have TextView as root element");
        }
    } else {
        tagLabel = new TextView(getContext());
        tagLabel.setPadding(mDecorator.getPadding()[0], mDecorator.getPadding()[1], mDecorator.getPadding()[2], mDecorator.getPadding()[3]);
        tagLabel.setTextSize(mDecorator.getTextSize());
    }
    updateCheckStatus(tagLabel, false);
    tagLabel.setText(tagString);
    tagLabel.setSingleLine();
    tagLabel.setGravity(Gravity.CENTER_VERTICAL);
    tagLabel.setEllipsize(TextUtils.TruncateAt.END);
    if (mDecorator.getMaxLength() != INVALID_VALUE) {
        InputFilter maxLengthFilter = new InputFilter.LengthFilter(mDecorator.getMaxLength());
        tagLabel.setFilters(new InputFilter[]{maxLengthFilter});
    }
    tagLabel.setClickable(true);
    tagLabel.setOnClickListener(this);
    addView(tagLabel, targetIndex);
    mInputView.requestFocus();
}

{% endhighlight %}

### 监听输入内容
对内容的判断在这里应该是比较重要的东西，同样非常简单，首先确定分隔符，这里默认使用回车，逗号来分割每一个标签项。

{% highlight java %}
@Override
public boolean onKey(View v, int keyCode, KeyEvent event) {
    if (event.getAction() == KeyEvent.ACTION_DOWN) {
        if (isKeyCodeHit(keyCode)) {
            if (!TextUtils.isEmpty(mInputView.getText().toString())) {
                generateTag(mInputView.getText().toString());
            }
            return true;
        } else if (KeyEvent.KEYCODE_DEL == keyCode) {
            if (TextUtils.isEmpty(mInputView.getText().toString())) {
                deleteTag();
                return true;
            }
        }
    }
    return false;
}

public void afterTextChanged(Editable s) {
    if (s.length() > 0) {
        if (isKeyCharHit(s.charAt(0))) {
            mInputView.setText("");
        } else if (isKeyCharHit(s.charAt(s.length() - 1))) {
            mInputView.setText("");
            generateTag(s.subSequence(0, s.length() - 1) + "");
        }
    }
}
{% endhighlight %}

### 删除标签
能加标签当然也能删除标签，考虑一下什么时候删除的是标签？当输入框为空的时候我们接着删除实际上急需要删除整个前面一项标签了。同时通过点击某一项标签，我们需要能删除当前选中的。

{% highlight java %}
private void deleteTag() {
    if (getChildCount() > 1) {
        removeViewAt(mCheckIndex == INVALID_VALUE ? indexOfChild(mInputView) - 1 : mCheckIndex);
        mCheckIndex = INVALID_VALUE;
        mInputView.requestFocus();
    }
}
{% endhighlight %}

### Layout排版
这里有一个问题和业务逻辑关系不大，但是和交互关系很大的的地方，就是layout中每一项view的位置摆放，理想情况下我们需要能让每个view自动填充在最后，同时可以自动换行，当删除某一项后，如果前面的空间足够，还要能够自动向前靠齐。

{% highlight java %}
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    int childTop = getPaddingTop();
    int childLeft = getPaddingLeft();
    final int count = getChildCount();
    for (int i = 0; i < count; i++) {
        View child = getChildAt(i);
        if (nullChildView(child)) continue;
        final int childWidth = child.getMeasuredWidth();
        final int childHeight = child.getMeasuredHeight();
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
        if (mCachedPosition.get(i, INVALID_VALUE) != INVALID_VALUE) {
            childTop += lp.topMargin + childHeight + lp.bottomMargin;
            childLeft = getPaddingLeft();
        } else if (childTop == getPaddingTop()) {
            childTop += lp.topMargin;
        }
        childLeft += lp.leftMargin;
        setChildFrame(child, childLeft, childTop, childWidth, childHeight);
        childLeft += childWidth + lp.rightMargin;
    }
}
{% endhighlight %}

要实现这种排版，方法其实很多，但大致原理是差不多的。在绘制view的时候我们需要手动计算每个view应该放在哪一行的什么位置，当前行放不下的时候，另起一行接着放。

## 小结
思路很重要，因为#@&！*（……&&……#%*&）%￥#@@%……&*（（*&……￥#@））  
写一个如图的流式标签生成控件大概就是就是这样，不能说很简单，但是其实也没那么难，理清思路和方案还是可以写出来的。
