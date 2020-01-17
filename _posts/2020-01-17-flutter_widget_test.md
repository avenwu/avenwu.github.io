---
layout: post
title: "行动起来，为你的Flutter项目添加组件测试"
description: ""
header_image: /assets/img/2020-01-17-01.png
keywords: "Flutter Widget测试"
tags: [Flutter]
---
{% include JB/setup %}
![img](/assets/img/2020-01-17-01.png)

* 目录
{:toc #markdown-toc}

## 背景
单元测试在很多平台下都是必要的存在，虽然由于各种原因真正落地的并不多。在flutter开发中，我们除了可以为模块添加基本的单元测试，也可以为Widget组件添加测试。

通过添加必要的case，可以保证我们的组件在迭代过程中，功能的”正确性“，一旦某次提交破坏了组件能力，即可便可在CI环节提前暴露出来。下面我们从`测试配置`，`测试API`,`问题处理`三个点介绍实际落地汇总遇到的问题。

## Widget测试配置

首次添加组件测试，可以根据[官方指南](https://flutter.dev/docs/cookbook/testing/widget/introduction)操作，按部就班配置即可：

```
1. 添加依赖库 flutter_test.
2. 创建/明确待测试的Widget.
3. 在test目录下，创建对应的测试文件，如test_widget.dart.
4. 使用`WidgetTester`接口，创建组件Build.
5. 使用`Finder`接口，查找组件.
6. 验证组件是否符合匹配要求，比如是否存在，可见性，文本内容等.

```
配置这块比较清楚，组件测试只需要flutter_test就够了：

```yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
```
如果测试存dart模块，可以继续添加test依赖：

```yaml
dev_dependencies:
  test: <latest_version>
```
## 操作组件的测试API
组件由于是可视化的一种布局元素，因此测试的时候需要根据规范来操作，下面我们结合笔者的项目进行详解。

笔者维护了一个Flutter的UI组件，在不断的迭代中，组件样式从最开始的1种，达到了9种。为了监控后续扩展的风险，开始引入组件测试。
> [add unit test and widget test](https://github.com/hacktons/convex_bottom_bar/pull/19)

根据前面我们提到过的组件测试引入流程，我们需要创建一个测试的dart文件：

```
├── pubspec.lock
├── pubspec.yaml
└── test
    └── widget_test.dart
```

这个测试文件，没有特别要求，默认放置在test文件夹下以test结尾，实现main函数即可。

截取了一个TestCase示例如下：

![img](/assets/images/appbar_widget_test.png)

简单整理下常见的API

| API接口        | 功能说明                                                     |
| -------------- | ------------------------------------------------------------ |
| testWidgets    | 创建组件测试的入口函数                                       |
| WidgetTester   | 可直接操作Widget的工具类，如点击操作tap，”渲染“组件pump      |
| find           | 查找组件元素的顶级函数，比如根据文案找组件，根据图标找组件   |
| findsOneWidget | 与expect配套使用，表示命中了一个组件，类似的还有findNothing等 |

详细的接口，可以通过源码在使用时查阅。

## 踩坑
踩坑根据不同组件可能会有差异，如果你的组件中使用了MediaQuery相关的API，那么在testWidgets创建时，需要考虑。不能只返回你的目标组件，而需要和App中类似，提供上层的相关支持。

比如可以添加如下代码来，解决MediaQuery和Row/Column引起的错误。

![img](/assets/images/widget_test_error_1.png)
![img](/assets/images/widget_test_error_2.png)

```dart
Widget material(Widget widget) {
  return Directionality(
    child: MediaQuery(
      data: MediaQueryData(),
      child: widget,
    ),
    textDirection: TextDirection.ltr,
  );
}
```
需要注意的是由于Flutter官方的文档，没有/也不太可能提供所有的问题集，因此在编写单元测试的时候，需要像编写demo一样，更加崩溃信息进行相应的用例调整。

添加完后，本地可以立即执行单元测试，远程仓库通过ci服务也可以在合适的时机触发构建测试：

![img](/assets/images/local_test.png)

![img](/assets/images/ci_pass.png)

## 参考

* [An introduction to widget testing](https://flutter.dev/docs/cookbook/testing/widget/introduction)