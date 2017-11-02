---
layout: post
title: "对异常收集的一些思考与优化"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/wuba/tango%E5%88%86%E6%9E%90%E6%97%A5%E5%BF%97%E6%B5%81%E7%A8%8B.png
keywords: "异常处理"
tags: [android]
---
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2017-09-11-02.png)


> 本文源于笔者内部分享整理而来;
相关插件的安装和使用请参考：[http://blog.hacktons.cn/tango/](http://blog.hacktons.cn/tango/)

## 1. 背景

在分析崩溃bug的时候，我们非常倚重调用栈日志，他在大多数情况下可以帮助开发者定位崩溃的逻辑链路，比如是哪个页面的控件出现了什么空指针之类的。

但是总有那么些个崩溃是没有这种有效日志的，没有办法定位崩溃源的样子真让人感到无助。

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
诸如此类信息不足以定位的bug还是很常见的。

## 2. 数据收集

为了方便定位问题，我们在早些时候时候写了一个Tango小模块，专门用来收集崩溃信息。
彼时我们根据需要定义了一套收集的内容和传输协议。

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

![Tango设计图][1]

经过一段时间的使用，这个模块有效的帮助我们定位解决了大量bug，现在几乎成了分析bug的一个必经流程，比如通过他定位bug的模块关系，从而分配负责修复的RD。

## 3. 协议扩展

由于业务的特殊性，在我们58App当中，存在大量的业务线，也存在很多载体相同但是动态承载分类业务的情况，比如我们的黄页大类下面有成百个自己分类，每个分类都可以进入对应的分类列表页面。
这个时候如果这种共性在载体页面发生了崩溃，我们如何定位他的业务类别呢？

基于此，我们想到扩展协议，让Tango手机必要的参数用于是被页面信息。通过梳理业务逻辑和思考，我们知道在Andorid中页面载体主要是Activity和Fragment，他们之间的跳转和承载必然会涉及到页面间的参数，而我们的业务线，实际上也是通过常规的intent和bundle进行数据传递的。因此我们的扩展非常明确，支持收集Activity的bundle[^IntentExtra]信息和Fragment的Bundle[^Fragment]信息。

![tango协议][2]

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
在收集的时候，需要注意兼容一下`android.support.v4.app.Fragment` 和 `android.app.Fragment`

## 4. 数据分析

过去我们分析上传的数据主要是人工，原因也很简单，为了让数据和bug能够一一吻合上，我们是直接把数据和崩溃信息一起上传到bugly[^Bugly]这边的。但是目前bugly并没有提供相关API供我们做数据分析和二次开发。所以我们的分析就是在bugly查询页面直接查看对应的日志，如果数据较多就复制出来通过json工具进行格式化后再阅读。

![手工分析日志流程][3]

这个流程基本上是可行的，但是也有他的缺点：

```
阅读日志的RD必须理解我们的的协议格式，起码要知道关键字段的含义。
```

现在我们的协议字段内容已经比较多了，如果继续按以前的老版本手工提取日志然后格式化查看的话就显得比较费劲了。因此我们设计并开发了专门用于提取Tango协议日志的工具 Tango助手[^TangoHelper]。

Tango助手基于Chrome Extension[^ChromeExtension] 技术开发, 可以在我们查阅bug详情时，自动提取并解析上传到bugly上的定制信息。 

![tango分析日志流程][4]
他的好处在于：

* RD分析bug日志的时候，直接看面板展示数据，协议是透明的，RD不用太关心；
* 整个提取过程是自动处理的，并可以通过UI界面将核心数据展示出来；

![可视化面板][5]

我们的数据面板有两块，通过设置项可以设置插件自动识别完日志后谭树提示框，这个信息只做了简单提取Activity，和格式化；
如果觉得自动弹窗比较烦人那么可以设置插件不弹窗（默认是不弹窗），这个时候日志提取完了，上哪去看呢？可以点击右上角的插件按钮，会弹出相对详细的内容。

如果说由于web页面大改版了或者bug什么的，导致插件没有自动提取到数据时怎么办呢，还可以手动提取，选中日志右键菜单选择`提取日志`就可以了，这个时候会再次识别选中内容并弹出提示面板。

## 5. 自动提取实现

好了，是时候讲一下我们的插件是如何工作的。

![tango插件工作流程][6]

* 首先当然是安装插件了，直接丢到chrome的插件面板内可以自动安装；
* 安装完后，我们可以去bugly查看bug了；
* 定位到某个具体的bug，进入详细信息页面，可以看到他的基本信息和调用栈信息；
* 切换到`跟踪日志`/`自定义日志`可以看到我们上传的日志，到这整个流程都是一样的；
* 此时插件在后台开始匹配当前的DOM数据，并且根据selector规则，去匹配我们的的日志所在的div区域，实际上这里最后是一个table；
* 解析命中的标签下的文本内容，如果不符合我们的协议格式，结束解析；
* 符合协议格式，解析完整json数据，并进行持久化操作，通过消息机制发送一个数据命中的message；
* 查看自动弹框开关，并展示弹框数据；
* 用户手动点击右上角插件按钮，展示详细数据面板。

基本上这就是我们的整个工作流程。如果懂js的话，基本可以照着整个思路直接实现了。

### 5.1 注意事项

这里提几个extension开发需要注意的一些点：

* background下定义的js是不可以直接操作前台页面的;
* 页面对外共享DOM树，但是不共享已经存在的js属性，成员等;
* content_scripts可以去做DOM交互；
* browser_action不可以主动触发，只能用户主动触发，也就是你不可以自己随意把插件的右上角面板弹出来；
* 发消息和手消息注意发送对象，避免首发不匹配；
* 插件可以国际化i18n，但是写起来有些麻烦；
* 访问页面请注意申请权限，不要用通配符随便申请所有页面，这样没必要也给使用者产生困扰。

所以我们的整体流程的工作其实是分散在不同js当中的，不同的面板触发也是不同逻辑。

讲了这么多，还不放点代码么？

### 5.2 代码示例

控制插件按钮的可用性，如果没有命中日志，我们希望按钮点击不生效，只有有数据的时候才启用。这个实现实际上是通过api动态设置按钮的状态实现的。

```js
function updateBrowserAction(enable) {
    if (enable) {
        chrome.browserAction.enable();
        chrome.browserAction.setTitle({title: i18n('browser_action_enable_tips')});
        chrome.browserAction.setBadgeText({text: "new"});
        chrome.browserAction.setBadgeBackgroundColor({color: '#f44336'});
    } else {
        chrome.browserAction.disable();
        chrome.browserAction.setTitle({title: i18n('browser_action_disable_tips')});
    }
}
```

你是怎么过滤的DOM数据? 这个使功能使我们的插件真正做到了自动解析，那么简单看下代码吧；

```js
function selectTangoLogElements() {
    console.log("select log elements...");
    // this elements sequence base on web page layout
    $('td span:nth-of-type(2)').each(function (index, element) {
        if (element.innerText.indexOf('Tango: ') !== -1) {
            var innerText = element.innerText;
            var logString = innerText.split('Tango: ')[1];
            console.log(logString);
            try {
                var tangoLog = JSON.parse(logString);
                tangoLog.timestamp = new Date().toUTCString();
                chrome.runtime.sendMessage({
                    parseTangoLog: true,
                    tangoLog: tangoLog
                }, function (r) {
                    console.log('send message', r);
                });
            } catch (e) {
                console.log('skip invalid log', e);
            }
        }
    })
}
```
实际上就是拦截了dom元素，然后根据我们提前观察，得出日志所在的dom树中的位置，然后就提取啦。

为什么你的弹框这么大，alert应该是很小很丑的以一个样子啊。这个确实比较麻烦， 笔者毕竟不是很懂前端的一套，想要我自定义一个alert是困难的。因此我的做法是往当前的dom里面插入一个div，这个div动态控制显示状态模拟出alert弹窗的效果就好了。

```js
$('#modal1').modal({
        dismissible: true, // Modal can be dismissed by clicking outside of the modal
        opacity: .5, // Opacity of modal background
        inDuration: 300, // Transition in duration
        outDuration: 200, // Transition out duration
        startingTop: '4%', // Starting top style attribute
        endingTop: '10%', // Ending top style attribute
        ready: function (modal, trigger) { // Callback for Modal open. Modal and trigger parameters available.
            console.log(modal, trigger);
        },
        complete: function () {
            console.log('Closed');
            unloadCSS('materialize-fonts-icon');
            unloadCSS('materialize-min');
        } // Callback for Modal close
    }
);
$('#modal1').modal('open');
```
示例就讲到这，最后我们在简单总结下。

## 7. 小结

一：PHP是最好的语言？  
二：插件里面主要使用的js，如果有些语法写的不正确，请原谅，我只懂这么多啦。  
三：对我们的异常收集有什么其他建设性的建议，欢迎指正。  


  [1]: http://7u2jir.com1.z0.glb.clouddn.com/wuba/tango%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E7%A4%BA%E6%84%8F.png
  [2]: http://7u2jir.com1.z0.glb.clouddn.com/wuba/tango%E5%8D%8F%E8%AE%AE.png
  [3]: http://7u2jir.com1.z0.glb.clouddn.com/wuba/%E6%89%8B%E5%B7%A5%E5%88%86%E6%9E%90%E6%97%A5%E5%BF%97%E6%B5%81%E7%A8%8B.png
  [4]: http://7u2jir.com1.z0.glb.clouddn.com/wuba/tango%E5%88%86%E6%9E%90%E6%97%A5%E5%BF%97%E6%B5%81%E7%A8%8B.png
  [5]: http://7u2jir.com1.z0.glb.clouddn.com/wuba/%E5%8F%AF%E8%A7%86%E5%8C%96%E9%9D%A2%E6%9D%BF.png
  [6]: http://7u2jir.com1.z0.glb.clouddn.com/wuba/tango%E6%8F%92%E4%BB%B6%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B.png


[^Bugly]: https://bugly.qq.com/v2/workbench/myapp

[^TangoHelper]: http://blog.hacktons.cn/tango/

[^ChromeExtension]: https://developer.chrome.com/extensions/getstarted

[^IntentExtra]: https://developer.android.com/reference/android/content/Intent.html

[^Fragment]: https://developer.android.com/reference/android/app/Fragment.html
