---
layout: post
title: "PNG压缩插件开发"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2017-06-29-01.png
keywords: "tinypng, pngquant, png"
tags: [图片]
---
{% include JB/setup %}

![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2017-06-29-01.png)

# 前言

为了压缩图片，经常会需要用到第三方的工具，比较知名的有tinypng和其他一些客户端软件，比如macOS下的ImageAlpha等；那么如果我们可以像新建文件一样在IDE直接就压缩图片，比如敲个快捷键 ctrl + N 直接搞定可不可以呢？

带着这个想法开始我们的插件开发。

# Biu

先看一下效果：

Biu是一款为IntelliJ IDEA设计的图片压缩插件，同时适用于基于IntelliJ开发的Android Studio。  
通过Biu您可以在IDE内“一键”压缩工程内的PNG图片资源。

以下是我们支持的一些操作：

* 支持右键压缩选中的PNG图片
* 支持批量压缩选中的文件夹内的所有PNG图片
* 支持通过文件选择器勾选所需压缩的PNG/文件夹
* 支持快捷键操作
* 支持压缩模式选择：[pngquant](https://pngquant.org/) 和 [tinypng](https://tinypng.com/)

目前支持如下64位操作系统：macOS，Win和Ubuntu

更多详情可以 访问我们项目站点 [http://avenwu.net/biu/](http://avenwu.net/biu/)

# 开发环境

进行开发前，可以根据以下表格做好开发环境配置：

配置项 | 说明 
------------ | -------------
PC/Mac | 一台开发用的电脑  
IntelliJ IDEA | 社区版/专业版都可以 
JDK| 无需强调，一般都安装了
API文档 | [Intellij SDK](http://www.jetbrains.org/intellij/sdk/docs/)
Swing文档 | [UI Swing](https://docs.oracle.com/javase/tutorial/uiswing/components/index.html)

基本上有了这几样就可以开发了，这里特意列了两个文档，不是开发必须的；但是对于不熟悉Swing和IntelliJ的开发者来说却是必须的。
IntelliJ支持多种开发语言，笔者选择的是Java来开发，爱折腾的你当然也可以选择其他语言，比如Groovy，Kotlin等，也是支持的，不过需要一些额外配置。

# UI与事件分发

记住在开发插件的时候，还是要注意线程问题的。虽然说大部分情况下就算你用错了线程，他也没那么容易像Android那样动不动就前台崩溃的。
我们的视图层是用Java Swing来做的，如果你在非UI线程操作视图会有一些警告。这里的UI线程指的是Swing用来专门做事件分发的线程 Event Dispatch Thread
简单的情况下可以用SwingWorker来处理，他封装了对UI线程事件分发的处理；值得一提的是SwingWorker是那种一看你就会用的API，因为Android里面的AsyncTask就是以他为原型开发的。

![]()

除了API接口，其内部实现也有很多神似的地方。
所以在异步操作之后要刷新UI的情况直接用这个现成的接口就可以。

# 异步任务

如果你不喜欢，想要深入操作细节，那么可以参考一下这个API文档：[https://docs.oracle.com/javase/tutorial/uiswing/concurrency/index.html ](https://docs.oracle.com/javase/tutorial/uiswing/concurrency/index.html )

其他所有Java的多线程也可以直接用。

## 代上码

下面结合具体的例子讲一下开发细节。

## 配置文件

在META_INF下的plugin.xml是插件的配置入口，这个和manifest类似，可以配置启动的一些组件，

{% highlight xml %}

<extensions defaultExtensionNs="com.intellij">
    <!-- Add your extensions here -->
</extensions>
 
 
<application-components>
    <!-- Add your application components here -->
</application-components>
 
 
<project-components>
    <!-- Add your project components here -->
</project-components>
<actions>
    <!-- Add your actions here -->
</actions>

{% endhighlight %}

不同组件作用不同，比如我们想在工具栏下新增一个按钮，就只需要在actions内新增一个action.
具体操作可以通过右键->New->Action, 也可以直接写配置

{% highlight xml %}

<action id="ToolPNGBatch" class="net.avenwu.tools.biu.action.ToolPNGBatchAction" text="Batch Compression"
        description="Open file tree dialog to choose images and compress together">
    <add-to-group group-id="ToolsMenu" anchor="last"/>
    <keyboard-shortcut keymap="$default" first-keystroke="ctrl alt B"/>
</action>

{% endhighlight %}

## 快速压缩单张图片实现

快速压缩单张图片，这个功能我们的定义是：

选中图片->右键->压缩 然后自动弹出Dialog完成展示压缩进度条等信息；

这样需要在菜单内加入一个子选项，核心在于确定菜单的group id，这个目前没有找到比较好的说明文档，只能逆行摸索。

{% highlight xml %}

<action id="OptionPNG" class="net.avenwu.tools.biu.action.OptionPNGMenuAction" text="Compress"
        description="Compress the selected png file directly">
    <add-to-group group-id="ProjectViewPopupMenu" anchor="first"/>
    <keyboard-shortcut keymap="$default" first-keystroke="ctrl alt P"/>
</action>

{% endhighlight %}

我们同时还定义了触发的快捷键 ctrl + alt + P 这个action绑定的实现类是net.avenwu.tools.biu.action.OptionPNGMenuAction,继承自AnAction实现actionPerformed等方法。
内部代码比较简单，就是根据选中文件是否为图片，选择性弹出压缩框；

{% highlight java %}

public void actionPerformed(AnActionEvent e) {
    System.out.println(System.getProperty("java.library.path"));
    VirtualFile virtualFile = Utils.targetFile(e);
    if (virtualFile != null) {
        QuickCompressDialog.newJDialog("Compress image", virtualFile).setVisible(true);
    }
}

{% endhighlight %}

## 批量压缩选择图片实现

批量压缩图片，这个功能我们的定义是：
打开文件选择框->选择多个图片或文件夹->弹出Dialog,展示所选文件->自动开始压缩，并更新列表    
类似的，我们这次需要在工具栏下面面新增一个子选项：

{% highlight xml %}

<action id="OptionPNG" class="net.avenwu.tools.biu.action.OptionPNGMenuAction" text="Compress"
        description="Compress the selected png file directly">
    <add-to-group group-id="ProjectViewPopupMenu" anchor="first"/>
    <keyboard-shortcut keymap="$default" first-keystroke="ctrl alt P"/>
</action>
{% endhighlight %}

同时还定义了触发的快捷键 ctrl + alt + B
批处理相对复杂以一些，主要是有大量操作表格的逻辑，和视图刷新。Swing控件中有一个JTable，和ListView有点类似，不过使用起来稍微麻烦一些。 

![jtable设计.png]({{ site.baseurl }}/assets/images/jtable设计.png)

这里面处理的时候需要注意排序问题，table是支持按列排序的，但是针对每一个的数据类型不同，排序规则需要自行定义，比如我们的压缩比数据源是浮点型，展示到table的时候做了一些转换，变为了带百分号的字符，这样导致默认排序采用的是文本排序，就不是我们预期效果。
同时每个单元格的样式问题也需要考虑，根据不同内容我们可能会希望将部分列左对齐，居中，右对齐等，比如要实现下图的排本，就需要单独处理单元格的render：

![批量压缩table.png]({{ site.baseurl }}/assets/images/批量压缩table.png)

# 文献资料

* [https://docs.oracle.com/javase/tutorial/uiswing/components/index.html](https://docs.oracle.com/javase/tutorial/uiswing/components/index.html)
* [http://www.jetbrains.org/intellij/sdk/docs/](http://www.jetbrains.org/intellij/sdk/docs/)
* [https://docs.oracle.com/javase/tutorial/uiswing/concurrency/index.html](https://docs.oracle.com/javase/tutorial/uiswing/concurrency/index.html)


