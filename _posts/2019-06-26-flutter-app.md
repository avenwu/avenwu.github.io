---
layout: post
title: "Flutter小试牛刀"
description: ""
header_image: /assets/img/2019-07-04-01.jpg
keywords: ""
tags: [Flutter]
---
{% include JB/setup %}
![img](/assets/img/2019-07-04-01.jpg)

* 目录
{:toc #markdown-toc}

## 背景
Flutter作为新晋`网红`，虽然还没有在我们项目中商用，但是热度已经有赶超ReactNative的趋势。
为了体验其开发效率和能力验证，我们将项目主界面中的个人页面进行Flutter重构。

> 为什么不选其他更复杂的界面呢？因为首页已经被重构了，没必要重复劳作。

![flutter-vs-native](/assets/images/flutter-vs-native.png)

## 构思
高度还原原版效果，可以对其进行截图，标注各部大小。为了简化这个过程，我们对大部分的模块只做预估，保证基本样子一致就行，不做像素级别的复制。
当然如果设计稿标注图还有也是可以的。

首先搭建静态页面，我们需要将整个页面进行拆解，适当划分模块可以进行合理的代码组织并便于后续维护。

![页面模块分解图](/assets/images/user-page-split-structure.png)

如图，可以大致分解为四块：
1. 头部
2. 头部运营位
3. 腰部运营位
4. 底部运营九宫格

## 界面抽象

基于前面的模块拆分，我们可以将每一块抽象为一个页面组件Widget，然后将四个组件按顺序组装在一起，就得到我们的效果图。
在学习了Flutter的技术文档之后，我们知道有很多基础的布局控件可以使用[Layout widgets](https://flutter.dev/docs/development/ui/widgets/layout)。

基本上通过该文介绍的组件我们就可以搭建出四个基础模块，下面逐一分析。

### 头部Widget

![native-header](/assets/images/native-header.png)

这里有一个底图是块大背景，然后有一个导航栏按钮，用户头像，昵称。由于Flutter中并不是每个组件都具备边距属性，因此为了精确实现效果，我们需要适当包裹一层边距，比如Container和Padding。

![native-header](/assets/images/header-stack.png)

```dart
_stackHeader(BuildContext context) {
  return Stack(
    children: <Widget>[
      Container(
        color: Colors.white,
        width: 480,
        child: Image.asset('assets/mycenter_head_bg.png', fit: BoxFit.cover),
      ),
      Container(
        margin: EdgeInsetsDirectional.only(top: 80, start: 15),
        child: InkWell(
          onTap: () => {
                Scaffold.of(context).showSnackBar(SnackBar(
                  content: Text('点击用户信息'),
                  duration: Duration(seconds: 2),
                ))
              },
          child: _userInfo(),
        ),
      ),
    ],
  );
}
```

### 头部运营控件

运营控件视觉上非常像一个表格，不过他的数量比较少，因此，我们可以直接通过`Row+Column`组合实现;

![native-header-card.png](/assets/images/native-header-card.png)

实现这个卡片表格，有几个点可以关注：
1. 卡片的周边投影
2. 圆角处理
3. 点击的Ripple效果
4. 屏幕高宽信息

*参考代码如下：*

```dart
Widget _gridCard(BuildContext context) {
  double cardMargin = 15;
  var width = MediaQuery.of(context).size.width - cardMargin * 2;
  return Card(
    margin: EdgeInsetsDirectional.only(
        start: cardMargin, end: cardMargin, bottom: cardMargin),
    shape: RoundedRectangleBorder(
        side: BorderSide(
            color: Colors.black12, width: 0.5, style: BorderStyle.solid),
        borderRadius: BorderRadius.all(Radius.circular(3))),
    elevation: 2,
    child: Column(
      children: <Widget>[
        Row(
          mainAxisAlignment: MainAxisAlignment.spaceAround,
          children: bean.icons
              .map((i) => _item(context,
                  source: i.image, text: i.text, width: width / 4))
              .toList(),
        ),
        Container(
          height: 0.5,
          margin: EdgeInsetsDirectional.only(start: 20, end: 20),
          color: Colors.black12,
        ),
        Row(
          children: <Widget>[
            _itemBottom(context,
                label: bean.left.label,
                text: bean.left.text,
                tips: bean.left.tips,
                width: (width - 0.5) / 2),
            Container(
              width: 0.5,
              height: 18,
              color: Colors.black12,
            ),
            _itemBottom(context,
                label: bean.right.label,
                text: bean.right.text,
                tips: bean.right.tips,
                width: (width - 0.5) / 2),
          ],
        ),
      ],
    ),
  );
}
```
### 腰部运营位
这个如果是轮播位还可以讲下，目前只是一个动态的图片位即可

![native-banner.png](/assets/images/native-banner.png)

```dart
// Banner
Widget _banner(BuildContext context) {
  return Container(
    height: 70,
    margin:
        EdgeInsetsDirectional.only(start: 15, top: 15, end: 15, bottom: 15),
    child: InkWell(
        onTap: () => {
              Scaffold.of(context).showSnackBar(SnackBar(
                content: Text('点击广告'),
                duration: Duration(seconds: 2),
              ))
            },
        child: CachedNetworkImage(
          placeholder: (context, url) => Container(
                color: Colors.black12,
                height: 70,
              ),
          errorWidget: (context, url, error) => Container(
                color: Colors.black12,
                height: 70,
              ),
          fit: BoxFit.cover,
          imageUrl: bean.bannerImage,
        )),
  );
}
```

### 底部运营九宫格

九宫格由于数量较多，可以通过GridView来实现，由于数量还是有限的，不存在无限列表的情况。

![native-grid.png](/assets/images/native-grid.png)

这里的几个关注点：
1. 横向个数控制
2. Item的高宽比
3. 分隔间隙

由于Flutter里面没有列表的Item点击事件，因此需要每个视图单独设置一个监听器；

```dart
@override
Widget build(BuildContext context) {
  return SliverGrid(
    gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
      crossAxisCount: 4,
      mainAxisSpacing: 1.0,
      crossAxisSpacing: 1.0,
      childAspectRatio: 1.2,
    ),
    delegate: SliverChildBuilderDelegate(
      (BuildContext context, int index) {
        IconBean data = this.response[index];
        return InkWell(
          onTap: () => {
                Scaffold.of(context).showSnackBar(SnackBar(
                  content: Text('点击：' + data.text),
                  duration: Duration(seconds: 2),
                ))
              },
          child: _item(source: data.image, text: data.text),
        );
      },
      childCount: this.response.length,
    ),
  );
}
```

## 细节优化

当整体搭建好了后，需要调整细节，比如点击效果和区域的调整，这种情况下会需要增加视图层级来包裹；
支持默认图的展示和背景色的处理，边距，也涉及到控件的包装。具体可以参考实现代码。

### 沉浸式

Android和iOS都有沉浸式效果，个人页面要支持沉浸式可以如下处理：

```dart
void main() {
  SystemChrome.setSystemUIOverlayStyle(SystemUiOverlayStyle(
       statusBarColor: Colors.transparent));
  runApp(MyApp());
}
```

### 圆角化
圆角化的方案有很多，常用的如下：

```dart
// 圆形头像 方案一 ClipOval
ClipOval(
  child: _cacheImage(
    'assets/personal_user_default_heade_img.png', this.avatarUrl),
),

_cacheImage(String placeholder, String source) {
  return CachedNetworkImage(
    placeholder: (context, url) => _defaultImage(),
    errorWidget: (context, url, error) => _defaultImage(),
    imageUrl: source,
    width: AVATAR_SIZE,
    height: AVATAR_SIZE,
  );
}
// 圆形头像 方案二 CircleAvatar
CircleAvatar(
  radius: 30,
  backgroundColor: Colors.white,
  backgroundImage: NetworkImage(this.avatarUrl),
),

// 圆角图片 ClipRRect
ClipRRect(
  borderRadius: BorderRadius.circular(10),
  child: _cacheImage(
    'assets/personal_user_default_heade_img.png', this.avatarUrl),
),
```
### 点击动效
点击效果，这里使用的是`InkWell`,对所有目标进行一次包裹

```dart
InkWell(
  onTap: () => {
        Scaffold.of(context).showSnackBar(SnackBar(
          content: Text('点击用户信息'),
          duration: Duration(seconds: 2),
        ))
      },
  child: _userInfo(),
);
```

![flutter-user-sample.gif](/assets/images/flutter-user-sample.gif)

## 小结

在开发常规页面是，Flutter还是非常趁手的，有一个差异点是，Flutter里面大量依赖`Widget`嵌套来实现效果，在开发Android时我们知道布局都是追求扁平化的。

这里嵌套是否会有性能问题？是一个可分析的点。
