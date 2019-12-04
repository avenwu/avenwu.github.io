---
layout: post
title: "InheritedWidget原理浅析与高阶应用"
description: ""
header_image: /assets/img/2019-12-03-01.jpeg
keywords: ""
tags: [Flutter]
---
{% include JB/setup %}
![img](/assets/img/2019-12-03-01.jpeg)

* 目录
{:toc #markdown-toc}

本文主要是为了探究两个事情：
1. InheritedWidget是如何实现的数据共享？
2. 基于他的实现的`provider`库，是怎么做的到局部刷新？

## Widget，Element，RenderObject?
在进行分析前，有必要同步一下认知，了解这三个术语的概念和作用。

* Widget =》 Describes the configuration for an Element.
* Element =》An instantiation of a Widget at a particular location in the tree.
* RenderObject =》An object in the render tree

Flutter在绘制这块采用了分层设计。很多总结的文章都会提到个概念：`渲染树Render Tree`。
Widget，Element，RenderObject其实都是围绕在渲染树这个概念下的。

* RenderObject是渲染树上的一个节点对象，负责paint，layout等具体的绘制流程
* Element持有RenderObject，他是Widget树里面Widget的具体实例
* Widget是概念上的组件，他描述了一个Element的配置信息

![](/assets/images/widget-element-renderobject.png)

## InheritedWidget是个啥
Widget作为开发者直接打交道的API，相对使用接触多。如Flutter里面的StatelessWidget和StatefullWidget。通过这两个基类我们可以继承实现各式各样的UI组件。

![](/assets/images/stateless-statefull.png)

两者的显著区别是是否存在State进行数据管理，有State的则可以进行一些更复杂的数据处理，并自动触发Widget的build。
![](/assets/images/state-flow.png)

如果跟一下这两个类，可知都是Widget的子类，
* StatelessWidget对应StatelessElement;
* StatefullWidget对应StatefullElement

InheritedWidget也是Widget的子类`InheritedWidget <= ProxyWidget <= Widget`。
对应InheritedElement。

他与前面两个的最大区别是InheritedWidget本身并不包含具体的widget，他只是一个代理的`壳`，用于`向下`传递数据。存在的目的是可以用于`规避数据层层传递`。
> Base class for widgets that efficiently propagate information down the tree.

InheritedWidget的使用比较简单，参考API文档就行。
```dart
class FrogColor extends InheritedWidget {
  const FrogColor({
    Key key,
    @required this.color,
    @required Widget child,
  }) : assert(color != null),
       assert(child != null),
       super(key: key, child: child);

  final Color color;

  static FrogColor of(BuildContext context) {
    return context.inheritFromWidgetOfExactType(FrogColor) as FrogColor;
  }

  @override
  bool updateShouldNotify(FrogColor old) => color != old.color;
}
```

## InheritedWidget的小魔法
到这里，我们对组件都有基础的认识。但是这根本不是重点好吗？只是前戏而已。

![](/assets/images/13_8457664.gif)

我们会用`InheritedWidget`，但是不知道他的内部原理，让人很空虚。特别是在优化局部刷新的时候，一度怀疑人生。

首先我们在明确下InheritedWidget的使用：
1. 继承InheritedWidget类
2. 重载updateShouldNotify，决定什么时候需要重新通知数据有变化
3. 提供获取实例的`of`接口

### of获取的对象是单例吗？

抛出第一个问题：
> 为什么of接口可以获取实例对象，是单例吗？

提供的工厂方法`of`能够获取指定类型对象是有前提的。共享数据的InheritedWidget，必须是父节点，兄弟节点不可以，子节点没有找到场景。 他不是单例，当当找不到类型的节点时返回空，直接使用会崩溃。因此需要开发者在逻辑上自行保证或者判空处理。
他的核心在于`inheritFromWidgetOfExactType`方法。

```dart
InheritedWidget inheritFromWidgetOfExactType(Type targetType, { Object aspect });
```

通过查阅文档说明可以总结一下核心点：

* 寻找父节点中类型匹配的最近节点并返回
* 将当期发起查询操作的BuildContext注册到对应的InheritedWidget数据监听中
* BuildContext其实就是调用者Widget本身所对应的Element实例
* 因此inheritFromWidgetOfExactType操作本质上建立了两个Widget的依赖关系

![](/assets/images/inherited-widget.png)

除此之外，这个方法也不是任意可以调用的，比如：
* iniState中不可以调用
* 可以直接或间接的在build方法中调用
* 类似也可以在 layout， paint的回调中执行，甚至didChangeDependencies中也可以

先看一下节点查找的实现代码, Element内部维护了一个依赖的哈希表，实现了时间复杂度为O(1)的查找。

![](/assets/images/inheritFromWidgetOfExactType.png)

从注释和demo验证来看，只要执行了inheritFromWidgetOfExactType方法；当InheritedWidget变化时，就会触发调用者的rebuild。

如果我们把inheritFromWidgetOfExactType换个名字,比如obtainAndSubscribe，是不是就较好理解了。

我们在重复一下他做的事情，核心就是两点：

* 获取指定类型的InheritedWidget
* 将自身添加到InheritedWidget的依赖表中

### 为什么调用of的widget会被刷新
第二个问题：
> 为什么调用of的widget会被刷新？（划重点，要考）

前面我们已经知道，获取实例的时候更新了依赖表。那么可以预见的到，Framework中一定有一个遍历依赖表，触发更新的循环操作：

![](/assets/images/notifyClients.png)

这个循环操作就是引起widgetrebuild的核心点。顺藤摸瓜，找到notifyClients的调用时机，一般就在build阶段，例如setState执行之后。
```
/// Notifies all dependent elements that this inherited widget has changed, by
/// calling [Element.didChangeDependencies].
///
/// This method must only be called during the build phase. Usually this
/// method is called automatically when an inherited widget is rebuilt, e.g.
/// as a result of calling [State.setState] above the inherited widget.
///
```

### 怎么让调用of的widget不被刷新

第三个问题：
> 怎么让调用of的widget不被刷新？（划重点，要考）

简单来说，没有办法不让widget刷新；要么就是不要调用inheritFromWidgetOfExactType。`provider`这个库也是通过使用了另一个api来获取数据，避免形成依赖关系。

关于InheritedWidget的挖掘，我们暂时到这就够了，下面进我们分析下provider的实现情况

![](/assets/images/20_8457054.gif)

## provider的局部刷新
状态管理的工具那么多，为什么偏偏要分析provider？

其实也没有什么特别原因，主要是这个库是官方首先推荐的，开发者是社区和Flutter团队成员，带有一定的官方色彩吧。如果读者对对provider完全不感冒，下面的分析可以忽略不看。

我们通过setState可以做数据驱动UI，但是如果情形复杂一些，出现了数据共享和局部刷新，光靠setState就很难实现了。provider本身是基于setState和InheritedWidget实现的。关于如何使用它，不是我们想要介绍的，请自行补齐知识：[Simple app state management](https://flutter.dev/docs/development/data-and-backend/state-mgmt/simple)

### Consumer做了什么
在使用provider的时候，我们需要将待观察的widget嵌套到Consumer中，但是他的源码其实也没啥东西，简单到`极致`
```dart
@override
Widget build(BuildContext context) {
  return Consumer<Foo>(
    builder: (_, foo, child) => FooWidget(foo: foo, child: child),
    child: BarWidget(),
  );
}
```
只需要一个实现一个buidler，可选的实现child部分。Consumer是StatelessWidget的子类，build方法中调用了我们传入的buidler，咋一看没有什么玄机，

![](/assets/images/consumer.png)

但是注意一个细节， provier参数是通过of方法获取的：
> Provider.of<T>(context)

![](/assets/images/6_3819879.gif)

如果你已经掌握了前面大篇幅的InheritedWidget源码分析，应该可以猜到这句话就是局部刷新的核心。

![](/assets/images/provider-of.png)

### provider如何通知数据更新
简单来说就是我们数据提供者需要继承ChangeNotifier，然后在修改了数据的时候调用一下notifyListeners方法，然后provider就会为我们完成剩下的工作。

可以看到provider利用了Framework提供的通知接口，这种观察模式说明，其内部一定帮我们完成了监听注册逻辑。
大致流程应该是这样：
> notifyListeners => 观察者listener执行 => widget build

下面我们就去找一下对应的实现逻辑。

> builder => delegate.initDelegate => startListening => listenable=》addListener=》setState

其中的listenable就是我们实现的ChangeNotifier数据提供者。
简单来说就是实例化以后就添加了监听listener，监听器的执行内容时调用setState更新buildCount

![](/assets/images/delegate-start-listen.png)

## 参考
* https://flutter.dev/docs/development/data-and-backend/state-mgmt/simple
* https://book.flutterchina.club/chapter14/element_buildcontext.html
* https://www.jianshu.com/p/988011994c22
* https://loveky.github.io/2018/07/18/how-flutter-inheritedwidget-works/


