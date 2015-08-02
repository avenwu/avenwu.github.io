---
layout: post
title: "AppBarLayout与ListView冲突，Footer丢失"
description: "conflicts between AppBarLayout and ListView Footer"
category: 
tags: [meterial design]
---
{% include JB/setup %}

Google新出的material design扩展包中，包含了几个很好用的控件，比如android.support.design.widget.CoordinatorLayout，android.support.design.widget.AppBarLayout
在实际使用中，发现有一些问题，如果滑动的控件为带Footer的ListView，那么在设置android.support.design.widget.AppBarLayout中子控件的时候不能设置

```
app:layout_scrollFlags="scroll|enterAlways"
```

此设置的意思是当滑动控件时，android.support.design.widget.AppBarLayout内对应的控件也跟着滑动，当往下货进来的时候，控件再次显示回来。

不想丢失Footer则可以将app:layout_scrollFlags设置为其他值；或者设置            


```
app:layout_collapseMode="pin"
```


并且目前CoordinatorLayout只支持RecyclerView与NestScrollView, 使用ListView，可以在Build.VERSION_CODES.LOLLIPOP之后手工设置

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    listView.setNestedScrollingEnabled(true);
}
```
         
{% highlight java %}
<?xml version="1.0" encoding="utf-8"?>
<android.support.design.widget.CoordinatorLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="?android:windowBackground">

    <android.support.design.widget.AppBarLayout
        android:id="@+id/app_bar_layout"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar">

        <android.support.v7.widget.Toolbar
            android:id="@+id/toolbar"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"
            android:background="?attr/colorPrimary"
            app:layout_collapseMode="pin"
            app:layout_scrollFlags="scroll|enterAlways"
            app:popupTheme="@style/ThemeOverlay.AppCompat.Light"/>

        <android.support.design.widget.TabLayout
            android:id="@+id/tabLayout"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:background="?colorPrimary"
            app:layout_collapseMode="pin"
            app:tabIndicatorColor="@android:color/white"
            app:tabSelectedTextColor="@android:color/white"
            app:tabTextColor="@color/tab_text_color"/>

    </android.support.design.widget.AppBarLayout>

    <android.support.v4.view.ViewPager
        android:id="@+id/pager"
        android:layout_width="wrap_content"
        android:layout_height="match_parent"
        app:layout_behavior="@string/appbar_scrolling_view_behavior"/>

    <android.support.design.widget.FloatingActionButton
        android:id="@+id/btn_compose_post"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="end|bottom"
        android:layout_margin="16dp"
        android:src="@drawable/ic_mode_edit_white_36dp"/>
</android.support.design.widget.CoordinatorLayout>

{% endhighlight %}