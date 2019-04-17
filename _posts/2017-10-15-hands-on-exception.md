---
layout: post
title: "Android端崩溃处理"
description: ""
header_image: /assets/img/showbug.png
keywords: "android, 异常处理"
tags: []
---
{% include JB/setup %}

笔者之前写过两篇类似主题的短文：《怎么看异常崩溃问题》，《对异常收集的一些思考与优化》，本文对两篇短文重新做了梳理使其更具完整性。

> PS： 全文大约7000字，快速阅读本文大概需要5分钟

## 1. 背景

长期以来，开发者都在和空指针，数组越界以及各种奇妙的疑难杂症斗智斗勇。业界也涌现了很多实用的异常收集工具，帮助我们追踪程序的崩溃信息，比如大名鼎鼎的 [fabric](https://get.fabric.io/)(前身是Crashlytics), 国内的老牌 [友盟](https://www.umeng.com/)，以及后起之秀 [bugly](https://bugly.qq.com/v2/)。

这些三方工具各有千秋，给我们解决崩溃问题提供了很大的帮助，但是也只是很大的帮助。我们基本能解决大部分崩溃问题，比如80%，但如果我们想让程序的稳定性更进一步，那就需要对剩下20%的问题发起攻坚战。

![Bug示意图](/assets/img/showbug.png)

### 1.1 移动应用崩溃现状

根据 [2016 移动应用质量大数据报告](http://blog.csdn.net/tencent_bugly/article/details/56015386) 的数据：

>Android应用行业整体崩溃率在`2.0%~3.6%`之间。其中视频、社交、音乐类应用的崩溃率较高，出行、新闻、儿童类应用的崩溃率较低。
根据产品规模日活（DAU）区间分析崩溃率，产品规模越大，崩溃率越低。DAU达`百万级别`的产品崩溃率平均在`1.5%`以下，对比各DAU区间崩溃率，游戏崩溃率均大于应用。

![中小规模产品崩溃率更高](http://oa5504rxk.bkt.clouddn.com/week33_report/10.png)

上面的数据是通过大的基数得出的均值，对于最求卓越的攻城师来说，这个指标是远远不够的。根据经验一般稳定的大厂app崩溃率都是小于等于`千分之一`左右的，另外好像有一个崩溃率`千分之一`，`千分之三`的数据统计，忘记在哪里看的，也找没相关素材:(

58同城Android端，通过提高代码质量以及bug分析效率等系列优化，将程序的崩溃率从`千分比`降低到了`万分比`，全量版本一般可以达到`万分之六`上下，当然这个数据也还有继续优化提升的空间。

这些优化措施中包含笔者下面要介绍的对崩溃收集的一点设计。

### 1.2 RD的困惑

在日常开发和版本迭代中，基本都是团队作战，通过分工各司其职。我们每版本都配备了负责上线事宜的RD，在轮值过程中每位上线RD或多或少都产生过`这个bug要找哪位RD去处理`的困惑。这其中很大一部原因是这种bug信息量不足以让我们定位问题，导致虽然我们有较为完整的bug收集处理流程，但总是每次都会遗留一些疑难杂症在表单中。

一般来说通过分析崩溃日志的调用栈关系，可以定位一些问题的发生位置，比如`业务代码`，`行号`之类的信息，然后按图索骥基本可以解决问题。但是还有一些bug是没法直接定位到业务代码的。来看几个例子，这是笔者选取的已经被修复的bug：

一个空指针，日志显示崩溃点为原生的FameLayout在对子元素layout的时候发生了空指针异常：
```
# main(1)
java.lang.NullPointerException
Attempt to invoke virtual method 'int android.view.View.getVisibility()' on a null object reference
1 android.widget.FrameLayout.layoutChildren(FrameLayout.java:531)
2 android.widget.FrameLayout.onLayout(FrameLayout.java:514)
3 android.view.View.layout(View.java:15695)
4 android.view.ViewGroup.layout(ViewGroup.java:5048)
//...
23 android.os.Handler.dispatchMessage(Handler.java:95)
24 android.os.Looper.loop(Looper.java:147)
25 android.app.ActivityThread.main(ActivityThread.java:5513)
26 java.lang.reflect.Method.invoke(Native Method)
27 java.lang.reflect.Method.invoke(Method.java:372)
28 com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:969)
29 com.android.internal.os.ZygoteInit.main(ZygoteInit.java:764)
```

另一个空指针，日志显示问题出在TextView控件measure的时候尝试获取文本行数，发生了空指针异常：

```
#1 main
java.lang.NullPointerException
Attempt to invoke virtual method 'int android.text.Layout.getLineCount()' on a null object reference
1 android.widget.TextView.onMeasure(TextView.java:6984)
2 android.view.View.measure(View.java:17935)
3 android.view.ViewGroup.measureChildWithMargins(ViewGroup.java:5698)
//...
13 android.widget.LinearLayout.measureChildBeforeLayout(LinearLayout.java:1723)
14 android.widget.TabWidget.measureChildBeforeLayout(TabWidget.java:165)
15 android.widget.LinearLayout.measureHorizontal(LinearLayout.java:1269)
16 android.widget.TabWidget.measureHorizontal(TabWidget.java:179)
17 android.widget.LinearLayout.onMeasure(LinearLayout.java:656)
18 android.view.View.measure(View.java:17935)
19 android.view.ViewGroup.measureChildWithMargins(ViewGroup.java:5698)
20 //...
39 android.view.View.measure(View.java:17935)
40 android.view.ViewRootImpl.performMeasure(ViewRootImpl.java:2412)
//...
```

这两个bug其实有一个共同点，就是日志很明确，但是`很难定位到与业务相关的代码逻辑`，如果强行分析的话，需要排查的面积很难控制。这直接导致这类问题分析效率低下，需要做大量的冗余无用功，最后还不一定能解决问题。

## 2. 换个思路

### 2.1 分析疑难杂症

> 这些疑难杂症怎么破？
这么“心碎”的问题，会不会也有别人遇到了，然后开启Google大法，幸运的时候可能在SO或其他什么博客上找到相见恨晚的答案，更多时候问题还是没法解决。

![Bug示意图](/assets/img/where-search-issues.png)

我们不妨先分析这些“疑难杂症”的日志，看是不是可以提炼出一些问题的特点，希望有助于我们提出针对性的解决方案。

* 日志信息很简单，可能只有寥寥几行；
* 与App的关联度几乎为零，都是些framework的信息；
* 崩溃原因很清楚，茫茫代码，就是不知道发生在何处；
* native异常，都是些云里雾里的日志；
* 。。。

既然日志信息这么匮乏，除了Google，深挖源代码，我们需要主动为崩溃补上一些信息，来辅助我们定位问题。

### 2.2 设立目标

让我们回归初衷：

> 导致我们bug分析困难是因为我们无法定位崩溃源。

对症下药，列出能有效帮助我们分析问题的元素

* 崩溃发生时的页面，如载体页是什么？
* 崩溃的进程？
* 崩溃时我们app展示了什么内容？
* 崩溃前的用户浏览轨迹？

如果有了这些信息bug是不是都能解决了？当然不可能，但是会大幅降低日志定位的门槛，间接提高消灭bug的成功率。从技术角度再确认下，我们的想法可行吗？

* 载体页，进程，页面收集可行，且接入成本较小
* 浏览轨迹可行，可能会有少量业务上的侵入
* 收集java层的常规异常可行（native层相对困难些）

总的来说，可以先做一些接入快，成本低的信息收集工作，根据使用的效果再进一步规划和完善。如此我们确定开发的基调和目标：

> 设计⼀个能够在应⽤程式崩溃时收集基础信息的⼯具

## 3. Tango Android设计

为了好记，我们给这个工具取名叫 `Tango`, 也寓意RD在分配bug，分析bug的时候可以像探戈一样更优雅一些。整个Tango分为两部分， 移动端收集数据和浏览器端解析数据，我们会分别介绍一下。

### 3.1 协议约定

没有规矩，不成方圆。既然我们已经确定了要收集哪些东西，那么有必要制定以下数据传输规格。

```json
{
    "process": "process name",
    "taskInfo": {
        "topActivity": "component name",
        "baseActivity": "component name",
        "origActivity": "optional component name"
    },
    "dump": {
        "activity": "activity name",
        "pluginName": "optional package name",
        "pluginClassName": "optional real activity name",
        "fragment#0": "fragment data",
        "deep": "optional hierarchy dump deep",
        "hierarchy": "optional view stack"
    }
}
```
这个基本就是代表我们会收集的数据，用json来组装数据主要是出于可读性考虑。下面有一个示意图，大致表示的是在android端tango的工作流程：
![Tango设计图](/assets/wuba/tango%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E7%A4%BA%E6%84%8F.png)

经过一段时间的使用，这个模块有效的帮助我们定位解决了大量bug，现在几乎成了分析bug的一个必经流程，比如通过他定位bug的模块关系，从而分配负责修复的RD。

### 3.2 协议扩展

由于业务的特殊性，在我们58App当中，存在大量的业务线，也存在很多载体相同但是动态承载分类业务的情况，比如我们的黄页大类下面有成百个子分类，每个分类都可以进入对应的分类列表页面。

> 如果这种共性在载体页面发生了崩溃，我们如何定位他的业务类别呢？

基于此，我们想到扩展协议，让Tango收集必要的参数用于识别页面信息。通过梳理业务逻辑和思考，我们知道在Andorid中页面载体主要是Activity和Fragment，他们之间的跳转和承载必然会涉及到页面间的参数，而我们的业务线，实际上也是通过常规的intent和bundle进行数据传递的。因此我们的扩展非常明确，支持收集Activity的bundle信息和Fragment的Bundle信息。

![tango协议](/assets/wuba/tango%E5%8D%8F%E8%AE%AE.png)

现在我们的协议有了一些变化：

```json
{
    "process": "process name",
    "taskInfo": {
        "topActivity": "component name",
        "baseActivity": "component name",
        "origActivity": "optional component name"
    },
    "dump": {
        "activity": "activity name",
        "extras": "Bundle[]",
        "fragments": [
          {
            "bundle": "Bundle[]",
            "fragment": "fragment name"
          }
        ],
        "pluginName": "optional package name",
        "pluginClassName": "optional real activity name",

        "deep": "optional hierarchy dump deep",
        "hierarchy": "optional view stack"
    }
}
```
### 3.3 注意事项

我们已经介绍了大致的搜集思路，那么在具体编码过程中也有一些需要额外注意的；

* 只收集必要的：收集数据的时候没必要保罗方方面面，集中精力解决核心问题，比如我们上线的时候并不收集view tree，但是开发环境下会收集。
* 避免耗时：不要尝试去做一些复杂的耗时业务。
* 兼容Framework：比如注意兼容一下`android.support.v4.app.Fragment` 和 `android.app.Fragment`。

## 4. Tango Chrome设计

这一节，讲一下插件开发，主要适用于Chrome下自动提取日志数据。

![tango chrome](/assets/img/2017-09-11-02.png)
### 4.1 数据分析
在设计插件之前，我们分析上传的数据主要是人工。原因很简单，在上报日志的时候，我们选用了成本最低的方案，直接把数据和崩溃信息一起上传到bugly。但是目前bugly并没有提供相关API供我们做数据分析和二次开发。所以我们的分析就是在bugly查询页面直接查看对应的日志，如果数据较多就复制出来通过json工具进行格式化后再阅读。

![手工分析日志流程](/assets/wuba/%E6%89%8B%E5%B7%A5%E5%88%86%E6%9E%90%E6%97%A5%E5%BF%97%E6%B5%81%E7%A8%8B.png)

在bugly上的具体操作位置如下：

![日志面板](/assets/img/bugly-custom-log.png)

这个流程基本上是可行的，但是也有他的缺点：

> 阅读日志的RD必须理解我们的的协议格式，起码要知道关键字段的含义

### 4.2 自动提取

现在我们的协议字段内容已经比较多了，如果继续按以前的老版本手工提取日志然后格式化查看的话就显得比较费劲了。因此我们设计并开发了专门用于提取Tango协议日志的工具 [Tango助手](http://blog.hacktons.cn/tango/)。

Tango助手基于Chrome Extension技术开发，可以在我们查阅bug详情时，自动提取并解析上传到bugly上的定制信息。 

![tango分析日志流程](/assets/wuba/tango%E5%88%86%E6%9E%90%E6%97%A5%E5%BF%97%E6%B5%81%E7%A8%8B.png)

他的好处在于：

* RD分析bug日志的时候，直接看面板展示数据，协议是透明的，RD不用太关心；
* 整个提取过程是自动处理的，并可以通过UI界面将核心数据展示出来；

![插件按钮](/assets/img/20170928-152448.png)

![展示面板](/assets/img/20170928-152628.png)


我们的数据面板有两块，通过设置项可以设置插件自动识别完日志后弹出提示框，这个信息只做了简单提取Activity和格式化；
如果觉得自动弹窗比较烦人那么可以设置插件不弹窗（默认是不弹窗），这个时候日志提取完了，上哪去看呢？可以点击右上角的插件按钮，会弹出相对详细的内容。

如果说由于web页面大改版了或者bug什么的，导致插件没有自动提取到数据时怎么办呢，还可以手动提取，选中日志右键菜单选择`提取日志`就可以了，这个时候会再次识别选中内容并弹出提示面板。

### 4.3 提取原理

好了，是时候讲一下我们的插件是如何工作的。

![tango插件工作流程](/assets/wuba/tango%E6%8F%92%E4%BB%B6%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B.png)

* 首先当然是安装插件了，直接丢到chrome的插件面板内可以自动安装（还不明白的话可以去这里看图文说明：[http://blog.hacktons.cn/tango/](http://blog.hacktons.cn/tango/)）；
* 安装完后，我们可以去bugly查看bug了；
* 定位到某个具体的bug，进入详细信息页面，可以看到他的基本信息和调用栈信息；
* 切换到`跟踪日志`/`自定义日志`可以看到我们上传的日志，到这整个流程都是一样的；
* 此时插件在后台开始匹配当前的DOM数据，并且根据selector规则，去匹配我们的的日志所在的div区域，实际上这里最后是一个table；
* 解析命中的标签下的文本内容，如果不符合我们的协议格式，结束解析；
* 符合协议格式，解析完整json数据，并进行持久化操作，通过消息机制发送一个数据命中的message；
* 查看自动弹框开关，并展示弹框数据；
* 用户手动点击右上角插件按钮，展示详细数据面板。

基本上这就是我们的整个工作流程。如果懂js的话，基本可以照着整个思路直接实现了。

### 4.4 注意事项

这里提几个extension开发需要注意的一些点：

* background下定义的js是不可以直接操作前台页面的;
* 页面对外共享DOM树，但是不共享已经存在的js属性，成员等;
* content_scripts可以去做DOM交互；
* browser_action不可以主动触发，只能用户主动触发，也就是你不可以自己随意把插件的右上角面板弹出来；
* 发消息和手消息注意发送对象，避免收/发不匹配；
* 插件可以i18n，但是写起来有些麻烦；
* 访问页面请注意申请权限，不要用通配符随便申请所有页面，这样没必要也给使用者产生困扰。

所以我们的整体流程的工作其实是分散在不同模块当中的，不同的面板触发也是不同逻辑。

## 5. 小结

最后我们再简单总结下。

对异常的处理从不同角度要考虑很多问题，比如突发异常的监控，每日崩溃率，异常分类归档，业务线划分等等。因此有些大厂也会做一套自身的崩溃处理体系。自研的好处是可定制定制性比较强，可以根据需求做出各种报表，监控体系。但是研发这样一套体系还是比较耗费人力的。因此更多的时候我们是利用第三方平台，自身则可以集中精力做业务等强关联的研发工作。

本文从实际的工作需求出发，聊了一些实用的收集技巧，如果读者对异常收集有何见解，欢迎交流/指正。 

* https://developer.chrome.com/extensions/getstarted
* https://developer.android.com/reference/android/content/Intent.html
* https://developer.android.com/reference/android/app/Fragment.html
* https://bugly.qq.com/v2/workbench/myapp
* http://blog.hacktons.cn/tango/

