---
layout: post
title: "聊聊Android手机的工程模式与实现"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-02.jpg
keywords: "工程模式"
description: "how to add hidden developer mode"
category: 
tags: [其他]
---
{% include JB/setup %}

![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-02.jpg)

## 背景

	使用Android设备的工程师都知道，设置界面里有一个工程模式--开发者模式，默认是关闭的，当连续点击版本信息后可激活开发者模式，在设置界面会新增一个选项；
	还有其他的一些其他隐藏彩蛋；
	类似原理，工程模式在商业应用内也被广泛使用，主要是方便开发者做一些动态操作；工程模式有很多实现方案，比如特定手势，特定账号，连续操作等，通过这些隐形操作触发工程模式的开启的条件；

## 管中窥豹，官方开发者模式

来看一下google官方对这个工程模式具体是如何处理的；

1. 打开设置页面，定位到**设置** -> **关于手机页面**，祭出hierarchyview

![setting-1.jpg]({{ site.baseurl }}/assets/images/setting-1.jpg)

![hierarchy-1.jpg]({{ site.baseurl }}/assets/images/hierarchy-1.jpg)


很快可以知道相关界面对应的Activity是com.android.settings.SubSettings,接下来渠道android source code中寻找相关线索；

很遗憾，SubSettings内没有什么新发现，

{% highlight java %}

package com.android.settings;

import android.util.Log;

/**
 * Stub class for showing sub-settings; we can't use the main Settings class
 * since for our app it is a special singleTask class.
 */
public class SubSettings extends SettingsActivity {

    @Override
    public boolean onNavigateUp() {
        finish();
        return true;
    }

    @Override
    protected boolean isValidFragment(String fragmentName) {
        Log.d("SubSettings", "Launching fragment " + fragmentName);
        return true;
    }
}
{% endhighlight %}
	
退回到com.android.settings包下逐一查找可疑目标；可以发现目标是DeviceInfoSettings

mDevHitCountdown变量记录连续点击的次数，每次点击后计数器自减，直到为0，则向sp内更新开发者模式的状态；

{% highlight java %}
else if (preference.getKey().equals(KEY_BUILD_NUMBER)) {
            // Don't enable developer options for secondary users.
            if (UserHandle.myUserId() != UserHandle.USER_OWNER) return true;

            final UserManager um = (UserManager) getSystemService(Context.USER_SERVICE);
            if (um.hasUserRestriction(UserManager.DISALLOW_DEBUGGING_FEATURES)) return true;

            if (mDevHitCountdown > 0) {
                mDevHitCountdown--;
                if (mDevHitCountdown == 0) {
                    getActivity().getSharedPreferences(DevelopmentSettings.PREF_FILE,
                            Context.MODE_PRIVATE).edit().putBoolean(
                                    DevelopmentSettings.PREF_SHOW, true).apply();
                    if (mDevHitToast != null) {
                        mDevHitToast.cancel();
                    }
                    mDevHitToast = Toast.makeText(getActivity(), R.string.show_dev_on,
                            Toast.LENGTH_LONG);
                    mDevHitToast.show();
                    // This is good time to index the Developer Options
                    Index.getInstance(
                            getActivity().getApplicationContext()).updateFromClassNameResource(
                                    DevelopmentSettings.class.getName(), true, true);

                } else if (mDevHitCountdown > 0
                        && mDevHitCountdown < (TAPS_TO_BE_A_DEVELOPER-2)) {
                    if (mDevHitToast != null) {
                        mDevHitToast.cancel();
                    }
                    mDevHitToast = Toast.makeText(getActivity(), getResources().getQuantityString(
                            R.plurals.show_dev_countdown, mDevHitCountdown, mDevHitCountdown),
                            Toast.LENGTH_SHORT);
                    mDevHitToast.show();
                }
            } else if (mDevHitCountdown < 0) {
                if (mDevHitToast != null) {
                    mDevHitToast.cancel();
                }
                mDevHitToast = Toast.makeText(getActivity(), R.string.show_dev_already,
                        Toast.LENGTH_LONG);
                mDevHitToast.show();
            }
        }
{% endhighlight %}

可以猜测，在显示开发者模式选项的页面必定会有相关读取该sp配置的逻辑；

{% highlight java %}

    /**
     * For Search.
     */
    public static final Indexable.SearchIndexProvider SEARCH_INDEX_DATA_PROVIDER =
            new BaseSearchIndexProvider() {

                private boolean isShowingDeveloperOptions(Context context) {
                    return context.getSharedPreferences(DevelopmentSettings.PREF_FILE,
                            Context.MODE_PRIVATE).getBoolean(
                                    DevelopmentSettings.PREF_SHOW,
                                    android.os.Build.TYPE.equals("eng"));
                }

                @Override
                public List<SearchIndexableResource> getXmlResourcesToIndex(
                        Context context, boolean enabled) {

                    if (!isShowingDeveloperOptions(context)) {
                        return null;
                    }

                    final SearchIndexableResource sir = new SearchIndexableResource(context);
                    sir.xmlResId = R.xml.development_prefs;
                    return Arrays.asList(sir);
                }

                @Override
                public List<String> getNonIndexableKeys(Context context) {
                    if (!isShowingDeveloperOptions(context)) {
                        return null;
                    }

                    final List<String> keys = new ArrayList<String>();
                    if (!showEnableOemUnlockPreference()) {
                        keys.add(ENABLE_OEM_UNLOCK);
                    }
                    if (!showEnableMultiWindowPreference()) {
                        keys.add(ENABLE_MULTI_WINDOW_KEY);
                    }
                    return keys;
                }
            };
}
{% endhighlight %}

至此google官方对开发者模式的显示，控制，基本完结；

* 可以发现，原理和实现都非常简洁，连续点击解锁，操作上简洁；
* 但是没有任何安全性，不适合在商业应用的使用；

## 他山之石，可以攻玉

虽然直接照搬上述模式不是很安全，但是解决问题的思路是一致的；

1. 设置合理的隐形开关条件，手势，指令；
2. 选择合适的输入页面，设置页面，登录页面；
3. 读写配置，刷新UI；

再介绍另外一个笔者实现的方案：手势开关；和开发者模式类似但原理稍有差别；

利用手势识别，如果在指定区域操作特定手势，那么就打开工程模式；

1. 首先约定用于解锁的手势，这个可以动态判断一个连续的系列手势，比如在5秒内先后在屏幕四个方向单击，双击；也可以生成一个手势文件，用作识别；
2. 连续指令很好理解，和官方的工程模式类似，只不过增加的多种方位，操作单击，双击结合，为了安全，可以吧操作指令写到c里面，判断的时候动态读取；
3. 手势文件，更简单些，通过GestureLibrary创建，识别手势；需要注意的是手势文件的读写操作，可以放在assets内，也可以对文件二进制数据加密存储，解密后读取；

具体代码就不上了，主要看思路；

