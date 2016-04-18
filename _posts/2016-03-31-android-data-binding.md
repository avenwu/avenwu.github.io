---
layout: post
title: "DataBinding踩坑Tips"
description: "Android Data Binding"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-31-04.jpg
keywords: "data binding"
category: 
tags: [android]
---
{% include JB/setup %}

![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-31-04.jpg)

## 前言
Android官方的MVVM技术Data Binding出来已经有一段时间了，这些天在开发YoYo Github使用后效果还不错，减少了之前大量的冗余代码，也完全取代了各种诸如ButterKnife之类的视图绑定工具。

## 踩坑--NavigationView的结合
在官方推荐的DrawerLayout侧滑菜单设计模式中，为了简化，统一风格，google为开发者提供了和DrawerLayout配套的NavigationView，一个NavigationView可以包含一个Header区域的独立Layout，以及一套menu菜单项，比如YoYo中的选项栏：

![device-2016-03-31-223548.png]({{ site.baseurl }}/assets/images/device-2016-03-31-223548.png)

如果想用DataBinding的思路来实现视图与数据绑定需要绕一个弯路；
{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>

<android.support.v4.widget.DrawerLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/drawer_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    tools:openDrawer="start">

    <include
        layout="@layout/app_bar_main"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

    <android.support.design.widget.NavigationView
        android:id="@+id/nav_view"
        android:layout_width="wrap_content"
        android:layout_height="match_parent"
        android:layout_gravity="start"
        android:fitsSystemWindows="true"
        app:menu="@menu/activity_main_drawer"
        tools:headerLayout="@layout/nav_header_main" />

</android.support.v4.widget.DrawerLayout>
{% endhighlight %}
这里面header部分的视图是单独提供的layout，实践发现如果要用DataBinding则不能在xml内指定该app:headerLayout属性(啰嗦一句，xml故意写的tools:headerLayout是预览作用，运行时并不会生效)；必须手动代码插入；
看一下这个单独的layout是怎么写的：

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>

        <import type="net.avenwu.yoyogithub.util.TimeUtils" />

        <import type="android.view.View" />

        <import type="android.text.TextUtils" />

        <variable
            name="user"
            type="net.avenwu.yoyogithub.model.User" />
    </data>

    <LinearLayout
        android:id="@+id/nav_header_container"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:background="@color/colorPrimary"
        android:gravity="bottom"
        android:orientation="vertical"
        android:paddingBottom="@dimen/activity_vertical_margin"
        android:paddingLeft="@dimen/activity_horizontal_margin"
        android:paddingRight="@dimen/activity_horizontal_margin"
        android:paddingTop="@dimen/activity_vertical_margin"
        android:theme="@style/ThemeOverlay.AppCompat.Dark">

        <RelativeLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="20dp"
            android:orientation="horizontal">

            <ImageView
                android:id="@+id/iv_avatar"
                android:layout_width="70dp"
                android:layout_height="70dp"
                android:scaleType="centerCrop"
                app:error="@{@drawable/ic_sort_black_24dp}"
                app:imageUrl="@{user.avatar_url}"
                app:roundAsCircle="@{true}"
                tools:src="@mipmap/ic_launcher" />

            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_centerVertical="true"
                android:layout_marginLeft="20dp"
                android:layout_toRightOf="@+id/iv_avatar"
                android:orientation="vertical">

                <TextView
                    android:id="@+id/tv_nick_name"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:paddingTop="@dimen/nav_header_item_vertical_spacing"
                    android:text="@{user.name}"
                    android:textAppearance="@style/TextAppearance.AppCompat.Headline"
                    tools:text="Aven Wu" />

                <TextView
                    android:id="@+id/tv_location"
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:drawableLeft="@drawable/ic_place_black_12dp"
                    android:drawablePadding="2dp"
                    android:paddingTop="@dimen/nav_header_item_vertical_spacing"
                    android:text="@{user.location}"
                    android:textAppearance="@style/TextAppearance.AppCompat.Caption"
                    android:visibility="@{TextUtils.isEmpty(user.location) ? View.GONE:View.VISIBLE}"
                    tools:text="Beijing, China" />

            </LinearLayout>

        </RelativeLayout>

        <TextView
            android:id="@+id/tv_email"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:drawableLeft="@drawable/ic_contact_mail_black_12dp"
            android:drawablePadding="8dp"
            android:layout_marginTop="@dimen/activity_vertical_margin"
            android:paddingTop="@dimen/nav_header_item_vertical_spacing"
            android:text="@{user.email}"
            android:visibility="@{TextUtils.isEmpty(user.email) ? View.GONE:View.VISIBLE}" />

        <TextView
            android:id="@+id/tv_blog"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:drawableLeft="@drawable/ic_public_black_12dp"
            android:drawablePadding="8dp"
            android:paddingTop="@dimen/nav_header_item_vertical_spacing"
            android:text="@{user.blog}"
            android:visibility="@{TextUtils.isEmpty(user.blog) ? View.GONE:View.VISIBLE}" />

        <TextView
            android:id="@+id/tv_join_time"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:drawableLeft="@drawable/ic_schedule_black_12dp"
            android:drawablePadding="8dp"
            android:paddingTop="@dimen/nav_header_item_vertical_spacing"
            android:text="@{`Joined on `+TimeUtils.formatTime(user.created_at)}" />

    </LinearLayout>
</layout>
{% endhighlight %}
几乎所有的控件和数据的绑定关系我们都成功的写在了xml里面，这样只需要在调用层直接为DataBing实现类设定我们定义的model即可；

{% highlight java %}
NavHeaderMainBinding binding = NavHeaderMainBinding.inflate((LayoutInflater) getSystemService(LAYOUT_INFLATER_SERVICE));
binding.setUser(data);
if (navigationView != null) {
	//需要注意的是，必须绑定完数据后才可以尝试获取并添加binding内的根视图到NavigationView中
    navigationView.addHeaderView(binding.navHeaderContainer);
}
{% endhighlight %}

developer api指南目前没有其他能解决解决NavigationView的办法，只能通过动态插入视图取巧绕过；

## 踩坑--Recyclerview.ViewHolder
除了简单的页面绑定，我们还可以绑定到Recyclerview的每一项元素，也就是ViewHolder这一层，这里也需要换一种绑定方式，因为ViewHolder是复用的；
先看一下ViewHodler现在要怎么写：
{% highlight java %}
public class ViewHolder extends RecyclerView.ViewHolder {
    private ViewDataBinding binding;

    public ViewHolder(View view) {
        super(view);
        binding = DataBindingUtil.bind(view);
    }

    public ViewDataBinding getBinding() {
        return binding;
    }
}
{% endhighlight %}
没错里面什么元素都没有，只有一个必须的ViewDataBinding基类，之所以用基类，是因为我们没办法动态知道每个Viewholder对应的DataBinding是哪一个，这屈居于ViewType；

同理整个ViewHolder关联的控件和数据关系我们也是写在了布局内：

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>

        <variable
            name="user"
            type="net.avenwu.yoyogithub.model.ShortUserInfo" />

        <variable
            name="listener"
            type="net.avenwu.yoyogithub.adapter.UserListAdapter.UserClickListener" />
    </data>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:background="@drawable/item_selector"
        android:onClick="@{listener.onClickItem}"
        android:orientation="horizontal"
        android:tag="@{user.login}">

        <ImageView
            android:layout_width="60dp"
            android:layout_height="60dp"
            android:layout_marginBottom="@dimen/activity_vertical_margin"
            android:layout_marginLeft="@dimen/activity_horizontal_margin"
            android:layout_marginTop="@dimen/activity_vertical_margin"
            android:scaleType="centerCrop"
            app:error="@{@drawable/ic_sort_black_24dp}"
            app:imageUrl="@{user.avatar_url}"
            app:roundAsCircle="@{true}"
            tools:src="@mipmap/ic_launcher" />

        <LinearLayout
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center_vertical"
            android:layout_marginLeft="10dp"
            android:orientation="vertical">

            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:singleLine="true"
                android:text="@{user.login}"
                android:textAppearance="?attr/textAppearanceListItem"
                tools:text="name" />

            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:ellipsize="end"
                android:paddingTop="5dp"
                android:singleLine="true"
                android:text="@{user.url}"
                android:textAppearance="?attr/textAppearanceListItemSmall"
                tools:text="home url" />
        </LinearLayout>
    </LinearLayout>

</layout>
{% endhighlight %}

有了ViewHodle和xml，现在把完整的Adapter写一下：

{% highlight java %}
public class UserListAdapter extends RecyclerView.Adapter<UserListAdapter.ViewHolder> {

    private final List<ShortUserInfo> mValues = new ArrayList<>();
    private UserClickListener mItemListener = new UserClickListener();

    @Override
    public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(parent.getContext())
                .inflate(R.layout.fragment_user_list, parent, false);
        return new ViewHolder(view);
    }

    @Override
    public void onBindViewHolder(final ViewHolder holder, int position) {
        holder.getBinding().setVariable(BR.user, mValues.get(position));
        holder.getBinding().setVariable(BR.listener, mItemListener);
        holder.getBinding().executePendingBindings();
    }

    @Override
    public int getItemCount() {
        return mValues.size();
    }

    public void addDataList(List<ShortUserInfo> list, boolean append) {
        if (append) {
            mValues.addAll(list);
        } else {
            mValues.clear();
            mValues.addAll(list);
        }
        notifyDataSetChanged();
    }

    public static class UserClickListener {
        public void onClickItem(View view) {
            if (view.getTag() instanceof String) {
                Toast.makeText(view.getContext(), (String) view.getTag(), Toast.LENGTH_SHORT).show();
            }
        }
    }
}
{% endhighlight %}

注意观察onBindViewHolder内的实现，我们需要显式设置model，并触发绑定；

## 小结
DataBinding的技术方案和常规实现有比较大的差异，因此在不熟悉的情况下要尽量多参考开发者api指南；

* [developer data binding guide](http://developer.android.com/tools/data-binding/guide.html)


