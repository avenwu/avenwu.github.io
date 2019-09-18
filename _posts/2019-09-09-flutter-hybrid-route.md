---
layout: post
title: "Flutter路由融合设计"
description: ""
header_image: /assets/img/2019-09-18-01.jpg
keywords: "打通Native协议路由与Flutter路由，实现原生协议通用兼容"
tags: [Flutter]
---
{% include JB/setup %}
![img](/assets/img/2019-09-18-01.jpg)

* 目录
{:toc #markdown-toc}

## 背景
开发Flutter项目的时候，我们通过Flutter原生的route实现页面间的导航，当我们将Flutter融合原生Android应用的时候后，遇到了一个问题，如何保证原生应用的路由协议在Flutter中同样生效？

本文基于这个场景，分享一下解决&设计思路。

## 两套路由
首先通过一小段代码，看一下原生和Flutter的路由差异

> Flutter路由

基本使用包括路由配置和页面切换，更多信息可以参考开发者文档[Navigation & routing](https://flutter.dev/docs/development/ui/navigation)

通过给`MaterialApp`组件的`routes`属性，配置路由表，key是字符串路由名，可以是随意的合法字符串，一般写成类似url的形式；value是Widget生成函数。 
```dart
MaterialApp(
  initialRoute: '/',
  routes: {
    '/': (context) => FirstScreen(),
    '/second': (context) => SecondScreen(),
  },
);
```

切换到一个定义好的页面，直接push即可。
```dart
onPressed: () {
  Navigator.pushNamed(context, '/second');
}
```

> App路由

原生App路由的话（也可以称作跳转协议），对不起，没有官方标准；各家都是自己的实现，也有些大厂开源了他们的路由协议，常规思路如下：
1. 维护路由配置表，根据配置表的生成可以是静态的，注解生成的，反射的，等等
2. 路由调度器，内部根据路由表，实现对配置和载体（一般是Acitivty或者说ViewController）的映射转换，包括一些必要参数的透传
3. 提供路由调用API接口，实现字符串跳转落地页，规避了开发者直接引用操作Activity/ViewController

同时为了实现外部吊起，路由协议一般都是基于Scheme的，这样，可以实现通过协议在外部吊起App的不同落地页。

比如我们可以定制一个性化的URI，通过不同字段代表不同含义。比如定一个协议规则如下：

> hoho://biz/page?params={}

![](/assets/images/app-route-demo.png)

为了打通App和Flutter之间的页面，同时保证App存量业务不受影响，我们需要让App的协议能够被Flutter内业务识别，同时可选的，我们可能需要让Flutter能够重载一些协议的落地页，比如说原理是App实现的相册，现在通过Flutter实现了，那么所有调用原相册协议的业务会很多，我们要做的业务无感知，就切换到新的Flutter页面上。

这里面的关键就是对App路由的拦截处理和Flutter路由的映射分发。

## Flutter路由插件

结合前面我们的分析，总结一下要做的事情，类似下图：

![](/assets/images/route-redirect.png)
1. `hoho://biz/gallery`可以在Flutter中识别
2. `hoho://biz/gallery`可以进一步落地到Flutter重载的页面

解决识别问题，可以开个一个Flutter的协议插件，专门用于调用App的老协议。为了对开发者友好，我们可以借鉴Navigator的API风格：
```dart
WBScheme.pushNamed('hoho://biz/gallery');
WBScheme.pushNamed('hoho://biz/feed');
```
如此，在需要跳转App原生页面的的时候，使用专用的路由API即可。

具体的协议转发将由我们的`WBScheme`在内部通过`MethodChannel`唤起Native逻辑，实现转发。

解决重载拦截问题，可块主要依赖于产检内部处理，调用层不用变化。
1. 在Flutter内调用，通过WBScheme内部管理，如果某协议是需要重载的，直接转发至route，不用向Native转发
2. 在Native内调用，通过App的路由，加油内部管理，如果某协议是需要重载的，转发至Flutter，不走Native原生跳转

可以发现，这里要分别处理在不同场景下的调用情况，并加以识别拦截，如果不希望设计的太复杂，并且也可以简化步骤一，即在Flutter内调用是不识别拦截，正常转发给Native，由native通过处理，该跳转的跳转，不该跳转的，抛回给Flutter跳转。

这样说起来可能有些绕口，看一张图概括一下：

![](/assets/images/route-redireect-call-flow.png)

如果不嫌麻烦，甚至可以Flutter的路由配置改成和App的路由配置一样，这样配置名称都一样

```dart
MaterialApp(
  initialRoute: '/',
  routes: {
    '/gallery': (context) => SecondScreen(),
    'hoho://biz/gallery': (context) => SecondScreen(),
  },
);
```

## 路由优化

在前面的介绍中，只是大致流程，很多细节没有介绍，其中也有一些可以注意的优化点
1. 使用Plugin的时候，如何保证Native分发值Flutter减少手写代码
2. 如果规避协议拦截所需要的明文配置/过多配置

这里有一个思路是，定好规则，通过dart自动生成代码来规避复杂的配置代码。


## 小结

在实现路由融合的过程比较简单，但是为了尽量减少配置代码，和不破坏Flutter页面逻辑，还是要考虑一番的。