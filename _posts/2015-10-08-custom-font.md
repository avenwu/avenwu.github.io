---
layout: post
title: "扩展LayoutInflator实现字体的全局设置"
description: ""
category: 
tags: [font]
---
{% include JB/setup %}

众所周知在Android系统上改变文本的字体，可以对TextView设置。这种常规的方法使用范围有限，如果需要全局设置就略显麻烦；

![字体设置汇总](http://7u2jir.com1.z0.glb.clouddn.com/device-2015-10-08-173228.png)

##字体准备
准备好需要用的字体文件，存放到assets内，比如：assets/font/

##常规设置
先看第一种方法，在合适的地方就可以直接通过setTypeface方法设置TextView的字体

{% highlight java %}
final Typeface typeface = Typeface.createFromAsset(getAssets(), "fonts/Oswald-Stencbab.ttf");
TextView textView = (TextView) findViewById(R.id.tv_label_font);
textView.setTypeface(typeface);
{% endhighlight %}

##封装-不写重复代码
上边的写法完全正确，但是如果每个页面需要设置字体都这样干的话是不行的，大量的读取assets文件势必对内存产生严重的浪费；

针对这个问题做一下优化：

{% highlight java %}
TypefaceUtils.setTypeface(this, (TextView) findViewById(R.id.tv_label_font_2), "fonts/Roboto-Bold.ttf");
{% endhighlight %}

可以看到通过TypefaceUtils类封装了对字体的设置和缓存操作

{% highlight java %}
public class TypefaceUtils {
    private static final Map<String, WeakReference<Typeface>> sCachedFonts = new HashMap<>();

    /**
     * A helper loading a custom font.
     *
     * @param assetManager App's asset manager.
     * @param filePath     The path of the file.
     * @return Return {@link android.graphics.Typeface} or null if the path is invalid.
     */
    public static Typeface load(final AssetManager assetManager, final String filePath) {
        synchronized (sCachedFonts) {
            try {
                if (!sCachedFonts.containsKey(filePath) || sCachedFonts.get(filePath).get() == null) {
                    final Typeface typeface = Typeface.createFromAsset(assetManager, filePath);
                    sCachedFonts.put(filePath, new WeakReference<Typeface>(typeface));
                    return typeface;
                }
            } catch (Exception e) {
                Log.w("Calligraphy", "Can't create asset from " + filePath + ". Make sure you have passed in the correct path and file name.", e);
                sCachedFonts.put(filePath, null);
                return null;
            }
            return sCachedFonts.get(filePath).get();
        }
    }

    public static void setTypeface(Context context, TextView textView, String filePath) {
        if (textView == null || context == null) return;
        textView.setPaintFlags(textView.getPaintFlags() | Paint.SUBPIXEL_TEXT_FLAG | Paint.ANTI_ALIAS_FLAG);
        textView.setTypeface(load(context.getAssets(), filePath));
    }
}
{% endhighlight %}


##全能大法-自定义Context
上面的方法基本可以满足大多数的需求，但是还存在一个问题，就是应用内的文本如果很多，并且需要全局替换为某一字体，那么一一设置文本是很不可取的办法。
解决这个问题一方面可以遍历view tree，把所有TextView子类都设置一遍，这个办法是可取的，但是用起来不是太方便；
通过分析Calligraphy项目，可以知道，作者在资源获取、视图加载的时候做了一些操作，本质上也是对所有TextView单独设置，但好处是不需要手工去遍历view tree，二时在view加载的时候被动触发；

这里大量的业务逻辑实际上是对LayoutInflator的理解和重载；

下面先看一下如何使用：

```
    @Override
    protected void attachBaseContext(Context newBase) {
        super.attachBaseContext(TypefaceContextWrapper.wrap(newBase));
    }
```
只需要在Activity内重载attachBaseContext方法，并设置我们自定义的Context就可以对该Activity内的所有文本进行设置，看起来非常简洁；

###自定义Context

首先继承ContextWrapper，只需要重载getSystemService，当获取的服务为LAYOUT_INFLATER_SERVICE时强制替换为自定义的TypefaceLayoutInflator；

{% highlight java %}
public class TypefaceContextWrapper extends ContextWrapper {
    private TypefaceLayoutInflator mInflater;

    public TypefaceContextWrapper(Context base) {
        super(base);
    }

    public static ContextWrapper wrap(Context base) {
        return new TypefaceContextWrapper(base);
    }

    @Override
    public Object getSystemService(String name) {
        if (LAYOUT_INFLATER_SERVICE.equals(name)) {
            if (mInflater == null) {
                mInflater = new TypefaceLayoutInflator(LayoutInflater.from(getBaseContext()), this);
            }
            return mInflater;
        }
        return super.getSystemService(name);
    }
}
{% endhighlight %}

###自定义LayoutInflator
接下来，重点的代码都在TypefaceLayoutInflator中；主要是重载cloneInContext，onCreateView，setFactory，setFactory2几个方法；

{% highlight java %}
public class TypefaceLayoutInflator extends LayoutInflater {
    public TypefaceLayoutInflator(Context context) {
        super(context);
    }

    public TypefaceLayoutInflator(LayoutInflater original, Context newContext) {
        super(original, newContext);
    }

    @Override
    public LayoutInflater cloneInContext(Context newContext) {
        return new TypefaceLayoutInflator(this, newContext);
    }

    @Override
    protected View onCreateView(View parent, String name, AttributeSet attrs) throws ClassNotFoundException {
        View view = super.onCreateView(parent, name, attrs);
        onViewCreatedInternal(view, getContext(), attrs);
        return view;
    }

    @Override
    protected View onCreateView(String name, AttributeSet attrs) throws ClassNotFoundException {
        View view = super.onCreateView(name, attrs);
        onViewCreatedInternal(view, getContext(), attrs);
        return view;
    }

    @Override
    public void setFactory2(Factory2 factory) {
        // Only set our factory and wrap calls to the Factory2 trying to be set!
        if (!(factory instanceof WrapperFactory2)) {
//            LayoutInflaterCompat.setFactory(this, new WrapperFactory2(factory2, mCalligraphyFactory));
            super.setFactory2(new WrapperFactory2(factory));
        } else {
            super.setFactory2(factory);
        }
    }

    @Override
    public void setFactory(LayoutInflater.Factory factory) {
        // Only set our factory and wrap calls to the Factory trying to be set!
        if (!(factory instanceof WrapperFactory)) {
            super.setFactory(new WrapperFactory(factory, this));
        } else {
            super.setFactory(factory);
        }
    }

    @Override
    public View inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot) {
        setPrivateFactoryInternal();
        return super.inflate(parser, root, attachToRoot);
    }

    private boolean mSetPrivateFactory = false;

    private void setPrivateFactoryInternal() {
        // Already tried to set the factory.
        if (mSetPrivateFactory) return;
        // Reflection (Or Old Device) skip.
//        if (!CalligraphyConfig.get().isReflection()) return;
        // Skip if not attached to an activity.
        if (!(getContext() instanceof Factory2)) {
            mSetPrivateFactory = true;
            return;
        }
        final Method setPrivateFactoryMethod = ReflectionUtils
            .getMethod(LayoutInflater.class, "setPrivateFactory");

        if (setPrivateFactoryMethod != null) {
            ReflectionUtils.invokeMethod(this,
                setPrivateFactoryMethod,
                new PrivateWrapperFactory2((Factory2) getContext(), this));
        }
        mSetPrivateFactory = true;
    }

    private static class WrapperFactory implements LayoutInflater.Factory {
        private final Factory mFactory;
        private final TypefaceLayoutInflator mInflater;

        public WrapperFactory(Factory factory, TypefaceLayoutInflator inflator) {
            mFactory = factory;
            mInflater = inflator;
        }

        @Override
        public View onCreateView(String name, Context context, AttributeSet attrs) {
            if (Build.VERSION.SDK_INT < Build.VERSION_CODES.HONEYCOMB) {
                try {
                    View view = mInflater.createCustomViewInternal(
                        null, mFactory.onCreateView(name, context, attrs), name, context, attrs);
                    onViewCreatedInternal(view, context, attrs);
                    return view;
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
            View view = mFactory.onCreateView(name, context, attrs);
            onViewCreatedInternal(view, context, attrs);
            return view;
        }
    }

    static void onViewCreatedInternal(View view, final Context context, AttributeSet attrs) {
        if (view instanceof TextView) {
            TypefaceUtils.setTypeface(context, (TextView) view, "fonts/RobotoCondensed-Regular.ttf");
        }
    }

    /**
     * Nasty method to inflate custom layouts that haven't been handled else where. If this fails it
     * will fall back through to the PhoneLayoutInflater method of inflating custom views where
     * Calligraphy will NOT have a hook into.
     *
     * @param parent      parent view
     * @param view        view if it has been inflated by this point, if this is not null this method
     *                    just returns this value.
     * @param name        name of the thing to inflate.
     * @param viewContext Context to inflate by if parent is null
     * @param attrs       Attr for this view which we can steal fontPath from too.
     * @return view or the View we inflate in here.
     */
    private View createCustomViewInternal(View parent, View view, String name, Context
        viewContext, AttributeSet attrs) throws Exception {
        // I by no means advise anyone to do this normally, but Google have locked down access to
        // the createView() method, so we never get a callback with attributes at the end of the
        // createViewFromTag chain (which would solve all this unnecessary rubbish).
        // We at the very least try to optimise this as much as possible.
        // We only call for customViews (As they are the ones that never go through onCreateView(...)).
        // We also maintain the Field reference and make it accessible which will make a pretty
        // significant difference to performance on Android 4.0+.

        // If CustomViewCreation is off skip this.
        if (view == null && name.indexOf('.') > -1) {
            if (mConstructorArgs == null)
                mConstructorArgs = ReflectionUtils.getField(LayoutInflater.class, "mConstructorArgs");

            final Object[] mConstructorArgsArr = (Object[]) ReflectionUtils.getValue(mConstructorArgs, this);
            final Object lastContext = mConstructorArgsArr[0];
            // The LayoutInflater actually finds out the correct context to use. We just need to set
            // it on the mConstructor for the internal method.
            // Set the constructor ars up for the createView, not sure why we can't pass these in.
            mConstructorArgsArr[0] = viewContext;
            ReflectionUtils.setValue(mConstructorArgs, this, mConstructorArgsArr);
            try {
                view = createView(name, null, attrs);
            } catch (ClassNotFoundException ignored) {
            } finally {
                mConstructorArgsArr[0] = lastContext;
                ReflectionUtils.setValue(mConstructorArgs, this, mConstructorArgsArr);
            }
        }
        return view;
    }

    private Field mConstructorArgs = null;

    @TargetApi(11)
    private static class WrapperFactory2 implements Factory2 {
        protected final Factory2 mFactory2;

        public WrapperFactory2(Factory2 factory2) {
            mFactory2 = factory2;
        }

        @Override
        public View onCreateView(String name, Context context, AttributeSet attrs) {
            View view = mFactory2.onCreateView(name, context, attrs);
            onViewCreatedInternal(view, context, attrs);
            return view;
        }

        @Override
        public View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
            View view = mFactory2.onCreateView(parent, name, context, attrs);
            onViewCreatedInternal(view, context, attrs);
            return view;
        }
    }

    @TargetApi(11)
    private static class PrivateWrapperFactory2 extends WrapperFactory2 {

        private final TypefaceLayoutInflator mInflater;

        public PrivateWrapperFactory2(Factory2 factory2, TypefaceLayoutInflator inflater) {
            super(factory2);
            mInflater = inflater;
        }

        @Override
        public View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
            try {
                View view = mInflater.createCustomViewInternal(parent,
                    mFactory2.onCreateView(parent, name, context, attrs),
                    name, context, attrs
                );
                onViewCreatedInternal(view, context, attrs);
                return view;
            } catch (Exception e) {
                e.printStackTrace();
            }
            return super.onCreateView(parent, name, context, attrs);
        }
    }
}

{% endhighlight %}

完整的简化版代码可以在这里获取

	https://github.com/avenwu/support
	
同时附上Calligraphy的项目地址
	
	https://github.com/chrisjenx/Calligraphy
	
##参考

* [https://github.com/chrisjenx/Calligraphy](https://github.com/chrisjenx/Calligraphy)