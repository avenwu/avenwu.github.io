---
layout: post
title: "Flutter相册优化指北"
description: ""
header_image: /assets/img/2019-07-08-01.jpeg
keywords: "性能优化 滑动监听 图片压缩 相册"
tags: [Flutter, 性能优化,滑动监听, 图片压缩, 相册]
---
{% include JB/setup %}
![img](/assets/img/2019-07-08-01.jpeg)

* 目录
{:toc #markdown-toc}

> 本文基于笔者在优化本地相册时的经历整理而来，阅读全文大约需要7分钟。

## 背景

Flutter作为当下跨平台开发技术栈中迅速崛起的黑马，我们在使用其开发的过程也是充满趣味。

> 一路采坑"一路爽"，性能优化"火葬场"

在58App中，我们有大量的发帖诉求，为了满足不同业务场景，甚至有多套风格不同的发帖&相册体系。比如发个房源信息，发个二手物品，发个部落帖子...

所以我们需要定制相册来满足业务：

1. 业务流程要求，如相册排序，展示UI
2. 相册交互，多选，单选，拍摄一条龙

Flutter官方API提供了类似原生相册选择器，这个不满足前面提到的要求。因此只能单独实现一把，然后原本很成熟的效果就遇到了性能问题。

> 在做相册优化的时候，Flutter官方版本还处于[v1.5.4-hotfix.2](https://flutter.dev/docs/development/tools/sdk/releases?tab=macos) 

## 自定义相册

我们看一下Native的相册长什么样：

![](/assets/images/device-2019-07-06-145047.png)



中规中矩的效果，支持九宫格显示相册，量体设计的尺寸和选中态处理。

在开发前先确定下，实现的思路。

1. 九宫格相册列表：通过GridView Widget实现
2. 选中状态：通过Stack Widget叠加视图实现，勾选角标和蒙层
3. 顶部bar，底部bar，适配拍摄，不是重点，可以先放一边

查询相册数据比较简单，通过Native的Plugin实现：

> 1. Flutter层以Channel向Native发起通信请求
> 2. Native查询系统相册目录后，得到相册列表，然后交回到Flutter层做UI展示

```dart
GridView.builder(
  gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 4,
    mainAxisSpacing: 4.0,
    crossAxisSpacing: 4.0
  ),
  padding: EdgeInsets.all(4.0),
  itemBuilder: _itemBuilder,
  itemCount: list.length,
);
```

列表的每一项是可复用的组件，加载本地图片需要用`Image.file( )`; 

运行后可以看到效果，整个相册比较飘[演示视频1](/assets/file/Record_2019-07-06-15-17-12.mp4)

![](/assets/images/Record_2019-07-06-15-17-12.gif)



**可以整理出几个问题**

> 1. 滑动后图片加载缓慢
> 2. 上下反复滑动后，图片出现重新加载
> 3. 快速滑动后，页面整体空白，等较长时间后才会开始出现图片

## 性能调优
思考列表的现象，直觉推测是GridView和Image之间的复用没处理好。以下是记录的一些主要优化思路和处理办法。

### 默认图占位处理

首先把占位图补充一下，希望在真正图片加载出来前，可以先看到一个默认图。注意默认图不要太大，同时Flutter不支持`.9.png`，如果想拉伸图片需要通过`centerSlice`，类似于`.9.png`拉伸。

```dart
FadeInImage(
  placeholder: AssetImage('assets/home_guess_like_item_icon.png'),
  image: FileImage(File(entity.path)),
  fit: BoxFit.cover,
);
```
占位图加上后，白屏现象基本解决，但是多次滑动过程还是会出现白屏。

### Widget组件复用
视图复用，这块其实问题不大，我们通过Builder模式使用GridView，官方介绍的是支持按需渲染视图的，注意关键词`created on demand`和 `appropriate for infinite`。可以得出在控件使用上是没有大问题的。

```
/// Creates a scrollable, 2D array of widgets that are created on demand.
///
/// This constructor is appropriate for grid views with a large (or infinite)
/// number of children because the builder is called only for those children
/// that are actually visible.
```

这里提到了一句话`those children that are actually visible`。其实默认并不这样的，不可见的也会被绘制，验证这一点很容易，你可以通过打印index和肉眼可见的元素作对比，可以发现index起始和结束是超出可见区个数的。
后面在分析源码的时候注意到了一个叫`cacheExtent`的参数。其默认值为250。

因此对这里来说，被绘制的"可见区"实际接近与：`屏幕高度+250*2`

![](/assets/images/gridview-draw-range.png)

### 图片内存复用

通过Flutter的`DevTool`和文档说明，我们可以验证Widget按需渲染是生效的。不过渲染仍然很慢啊，也有可能是视图复用但是内存没有复用。这个需要分析Image的源码实现。

我们使用的是本地图片文件，相关代码可以查看`FileImage.dart`。一顿操作后可以总结以下两点

1. 图片解码是直接将图片的字节数组进行解码的，这个过程**没有预处理**和**缩放策略**
2. 解码工作不在dart层，而是通过PaintingBinding层层封装，最终调用的是C层函数解码
3. 没有看到复用内存的逻辑

```dart
Future<ui.Codec> _loadAsync(FileImage key) async {
    assert(key == this);
    final Uint8List bytes = await file.readAsBytes();
    if (bytes.lengthInBytes == 0)
      return null;
    return await PaintingBinding.instance.instantiateImageCodec(bytes);
}
```

前两点没什么好纠结的，第三点深挖一下，可以知道：

> 图片提供器的父类`ImageProvider`具备基本的内存复用能力

内存复用通过`ImageCache`实现，具体对象为`PaintingBinding.instance.imageCache`。

它通过`putIfAbsent`方法实现了缓存获取和设置的逻辑，key则是前面已经看到的，由每个`ImageProvider`子类实现的`obtainKey`方法提供。

更具体的，针对FileImage，他的key就是FileImage本身，大家可以关注下这个类的相等判断，他重载了`==`操作符和`hashcode`函数。

```dart
key.then<void>((T key) {
  obtainedKey = key;
  final ImageStreamCompleter completer = PaintingBinding.instance
    .imageCache.putIfAbsent(key, () => load(key), onError: handleError);
  if (completer != null) {
    stream.setCompleter(completer);
  }
}).catchError(handleError);
```

**通过这些分析，我们可以确定一点内存复用时存在的，但是为什么么效果不好呢？**

为了验证内存复用情况，我们祭出Debug调试一下，最终发现了真相：

> 1. 单张图片内存占用巨大，甚至达到20MB。
> 2. 图片缓存是有阈值的，大图片与缓存数成反比，间接影响缓存命中率。

默认缓存图1000张，默认最大内存缓存为100MB,两者触发其一，图片就会根据`LRU`策略进行更换。

```dart
const int _kDefaultSize = 1000;
const int _kDefaultSizeBytes = 100 << 20; // 100 MiB
```

```dart
// Remove images from the cache until both the length and bytes are below
// maximum, or the cache is empty.
void _checkCacheSize() {
  while (_currentSizeBytes > _maximumSizeBytes || _cache.length > _maximumSize) {
    final Object key = _cache.keys.first;
    final _CachedImage image = _cache[key];
    _currentSizeBytes -= image.sizeBytes;
    _cache.remove(key);
  }
  assert(_currentSizeBytes >= 0);
  assert(_cache.length <= maximumSize);
  assert(_currentSizeBytes <= maximumSizeBytes);
}
```
如果你的应用不需要这么大的缓存，可以自行调整。

### 大图按需剪裁

通过前面内存的分析，我们知道默认加载`Image.file`时，没有任何优化措施。本地相册的图片一般是相机拍摄的，目前高分辨率的手机满大街都是。

> 假设拍摄了一张1280 x 2560的图片，一个像素使用4字节的话，把它加载到内存中，可以达到1280 x 2560 x 4bytes = 12.5MB

`100/12.5=8`也就是说8张图片就把默认的内存缓存沾满了。如果图片再大点，8张可能都不到，假设我们做九宫格，那么展示两行半以后就会触发缓存的更新。

解决这个问题的思路很简单，我们想办法对图片展示和渲染进行干预，提供剪裁能力。

#### 缩放策略（一）

再次分析`FileImage`的源码，发现有一个缩放参数`scale`，他的作用是绘制图片时按指定的缩放比缩放图片，感觉功能有点像。

```dart
/// The scale to place in the [ImageInfo] object of the image.
final double scale;
```

不过实际使用后会发现，这个参数对我们这个问题没有什么帮助，原因从源码深入分析可以知道。scale只控制了`绘制的缩放`，内存占用并没有减少。

他的逻辑可以概括如下：

> 1.原始图片文件 => 2.原始图片字节数组 => 3.codec图片编码 => 4.Image实例 => 5.画布

我们需要第三步或者第四步之后的内存占用真正减小。

这里我们可以考虑定义一个`ResizeImage`实现图片的像素缩放，基于我们实现的自缩放Widget，图片内存确实减少了。

```dart
Future<Codec> _loadAsync(ResizeFileImage key) async {
  assert(key == this);
  final Uint8List bytes = await file.readAsBytes();
  if (bytes.lengthInBytes == 0) return null;
  final Uint8List resizeBytes = await resize(bytes, file.path);
  return resizeBytes;
}
Future<Uint8List> resize(Uint8List bytes, String name) async {
  extendImage.Image image = await extendImage.decodeImage(bytes);
  extendImage.Image thumbnail = extendImage.copyResize(image, width: 400);
  return Uint8List.fromList(extendImage.encodeNamedImage(thumbnail, name));
}
```

在单图测试情况下效果还不错，但是引入到相册后，大量图片加载情况下，弊端出现了。 

> resize相对耗时，大量图片加载会导致页面不流畅

自定义的Widget中计算任务过重会导致页面性能，这里计算任务是`resize`处理，需要提前做两次转码，导致非常耗CPU。

#### Framework分析

通过深入分析Flutter相关图片绘制源码，我们发现目前系统根本不提供指定高宽来解码图片。这真是令人着急的系统API啊。

最后在Github上，我们我们发现Flutter社区在19年5月有人提出了图片的解码尺寸问题，并且在5.9号合并到了主分支。https://github.com/flutter/engine/pull/8596/files

> Expose API to decode images to specified dimensions #8596
>
> Merged [iskakaushik](https://github.com/iskakaushik) merged 14 commits into [flutter:master](https://github.com/flutter/engine) from [iskakaushik:expose-resizing-api](https://github.com/iskakaushik/engine/tree/expose-resizing-api) on May 9

核心是painting.dart提供了参数，支持指定高宽。类似于Android 里面的BitmapFactory处理。

```dart
Future<Codec> instantiateImageCodec(Uint8List list, {
  double decodedCacheRatioCap = 0,
  int targetWidth,
  int targetHeight,
})
```

同时另外几个类似方法也一并进行了支持。进一步确认该优化的版本信息，至少需要将Flutter升级到2019.5.9号之后的版本。

```shell
aven-mac-pro-2:work aven$ flutter --version
Flutter 1.5.4-hotfix.2 • channel stable • https://github.com/flutter/flutter.git
Framework • revision 7a4c33425d (9 weeks ago) • 2019-04-29 11:05:24 -0700
Engine • revision 52c7a1e849
Tools • Dart 2.3.0 (build 2.3.0-dev.0.5 a1668566e5)
```

Flutter发布版本包括四个渠道, 建议使用稳定版本，有时候体验新API，也可以使用Beta版本。

| Version/Stable                                               | Ref     | Release Date |
| ------------------------------------------------------------ | ------- | ------------ |
| [v1.5.4-hotfix.2](https://storage.googleapis.com/flutter_infra/releases/stable/macos/flutter_macos_v1.5.4-hotfix.2-stable.zip) | 7a4c334 | 5/8/2019     |
| [v1.2.1](https://storage.googleapis.com/flutter_infra/releases/stable/macos/flutter_macos_v1.2.1-stable.zip) | 8661d8a | 2/27/2019    |
| [v1.0.0](https://storage.googleapis.com/flutter_infra/releases/stable/macos/flutter_macos_v1.0.0-stable.zip) | 5391447 | 12/5/2018    |

| Version/Beta                                                 | Ref     | Release Date |
| ------------------------------------------------------------ | ------- | ------------ |
| [v1.6.3](https://storage.googleapis.com/flutter_infra/releases/beta/macos/flutter_macos_v1.6.3-beta.zip) | bc7bc94 | 5/31/2019    |
| [v1.5.4-hotfix.2](https://storage.googleapis.com/flutter_infra/releases/beta/macos/flutter_macos_v1.5.4-hotfix.2-beta.zip) | 7a4c334 | 5/3/2019     |
| [v1.5.4-hotfix.1](https://storage.googleapis.com/flutter_infra/releases/beta/macos/flutter_macos_v1.5.4-hotfix.1-beta.zip) | 09cbc34 | 5/1/2019     |
| [v1.5.4](https://storage.googleapis.com/flutter_infra/releases/beta/macos/flutter_macos_v1.5.4-beta.zip) | b593f51 | 4/27/2019    |


#### 缩放策略（二）

下面尝试使用新的Flutter API实现缩放能力，升级版本到`1.6.3`:


```shell
aven-mac-pro-2:work aven$ flutter --version
Flutter 1.6.3 • channel beta • https://github.com/flutter/flutter.git
Framework • revision bc7bc94083 (6 weeks ago) • 2019-05-23 10:29:07 -0700
Engine • revision 8dc3a4cde2
Tools • Dart 2.3.2 (build 2.3.2-dev.0.0 e3edfd36b2)
```

同时改造一下我们的`ResizeFileImage`:

```dart
Future<Codec> _loadAsync(ResizeFileImage key) async {
  assert(key == this);
  final Uint8List bytes = await file.readAsBytes();
  if (bytes.lengthInBytes == 0) return null;
  return await instantiateImageCodec(bytes,
      targetHeight: this.targetHeight, targetWidth: this.targetWidth);
}
```
这个效果就非常好了，结合调试工具，可以看到图片高宽解码后确实发生了变化。如此ImageCache压力得到了极大的释放，这里顺便看一下ImageCache对内存大小的计算，前面我们提到的图片张数预估也可以在这里得到验证。

*相关源码如下：*

```dart
void listener(ImageInfo info, bool syncCall) {
  // Images that fail to load don't contribute to cache size.
  final int imageSize = info?.image == null ? 0 : info.image.height * info.image.width * 4;
  final _CachedImage image = _CachedImage(result, imageSize);
  // If the image is bigger than the maximum cache size, and the cache size
  // is not zero, then increase the cache size to the size of the image plus
  // some change.
  if (maximumSizeBytes > 0 && imageSize > maximumSizeBytes) {
    _maximumSizeBytes = imageSize + 1000;
  }
  _currentSizeBytes += imageSize;
  final _PendingImage pendingImage = _pendingImages.remove(key);
  if (pendingImage != null) {
    pendingImage.removeListener();
  }

  _cache[key] = image;
  _checkCacheSize();
}
```

### 可视区优化

完成了上面的优化之后，整个相册滑动其实已经很流畅了。但是作为精益求精的话，我们可以在进行一些优化。在页面Fling一段之后停下，页面可视部分的图片加载没有立刻得到相应。需要等一会儿才行。

这里有两个思路优化

> 1.可视区优化

结合前面已经分析过的视图复用，我们把默认的cacheExtent改小一点，比如不可见的时候之渲染而外一行图片。

> 2.动态计算位置

由于Flutter是声明式布局，state改变后Widget Tree都会重新build，所以前后滑动过程中要谨慎处理state，这里我们再次实现了一个Widget，用于懒加载图片`LazyLoadImage`。

核心思路是，引入一个加载状态，通过构造函数传入不同的时延，实现滑动过程中延时甚至不加载图片，滑动停止后快速加载图片。

```dart
LazyLoadImage({
  Key key,
  this.delay = 100,
  @required this.placeholder,
  @required this.image,
  this.width,
  this.height,
  this.fit,
  this.alignment = Alignment.center,
  this.repeat = ImageRepeat.noRepeat,
  this.filterQuality = FilterQuality.low,
})
```

这里你可能会觉得思路简单应该很好实现，其实并不是这样的。感兴趣的读者可以尝试下懒加载和可视区的判定，这是本节的两个难点，提示：

1. 延迟加载，可以引入图片的状态机
2. 可视区检测可以结合HitTestResult

### Jank优化

经历了前面的优化措施，我们的相册流畅性和内存使用有了显著提高，但是通过与Native原生相册对比体验，感觉有的还是有点细微差距。比如起始滑动有瞬间“粘滞感”，fling后很顺滑。此时我们需要精细化分析，可以通过Flutter的性能分析工具进行测量。

![](/assets/images/timeline-UI-GPU.gif)


基于我们的分析，归纳起来做了以下方面调整：

1. 对象创建耗时，尽可能去除无用代码，比如未被使用的动画，bean的创建
2. 默认占位图的调整，可以考虑用MemoryImage或色值
3. 布局优化，减少Widget数量
4. Log移除，debugLog也是耗时的，特别是在生命周期内会经常触发的日志
5. 用release包做最后体验，`release，release，release`，重要的事情说三遍

优化完毕后打出Relase的包效果就可以和原生实现的相册媲美了。

![](/assets/images/Record_2019-09-05-17-45-24.gif))

## 闲鱼Flutter

在进行相册优化的时候，查阅过很多社区的文章，几乎没有真的相册优化的。找到的唯一技术分析文章是闲鱼的。

> 文章信息透露相册使用纹理实现，借助Native来实现图片的完整解码和渲染，感兴趣的读者可以查看原文：[一个优秀的可定制化Flutter相册组件，看这一篇就够了](https://www.yuque.com/xytech/flutter/ey982d)

这个思路和我们的方向不太一致，文章也没有给出具体的实现和优化，整体来说这篇文章并没有太多借鉴意义。

闲鱼宣传Flutter这块名声还是很响亮的，而且各种技术分享文章也很多，这里我们具体看下闲鱼到底哪些地方使用了Flutter构建业务。

![](/assets/images/xinayu-zhihu.png)

逆向了一下闲鱼App，虽然有引入flutter，但是笔者并没有发现多少使用Flutter的地方，

> 闲鱼的相册并不是Flutter实现的，意不意外？

这也许能解释为什么文章没有贴出任务实例代码，有可能这是闲鱼内部的迭代，相册根本没有用Flutter实现或者有灰度策略吧，笔者正好被排除在外了。下面是闲鱼相册实现载体和页面布局情况：

![](/assets/images/xianyu-gallery-activity.png)

![](/assets/images/xianyu-gallery-viewtree.png)

那么闲鱼到底在哪里使用了Flutter呢？最后在一篇文章找到了线索， `部分贴子详情`

![](/assets/images/xianyu-introduce.png)

代码层面，有几个看起来是Flutter的载体：

- com.idlefish.flutterbridge.flutterboost.FishFlutterActivity
- com.taobao.idlefish.flutterboost.BoostFlutterActivity
- com.taobao.flutterchannplugin.FlutterWrapperActivity

也许再不远的将来，可以看到闲鱼文章中提到的相册在闲鱼App的落地，这样也可以体验一下他的效果。

## 小结

Flutter在快速的发展，现在遇到的问题，相信经过足够的时间，业内会涌现出各种不同的解决方案。

## 参考
1. [[WIP] Expose API that allows for network resizing images](https://github.com/flutter/flutter/pull/31164)
2. [Expose API to decode images to specified dimensions](https://github.com/flutter/engine/pull/8596)
3. [flutter.rendering.viewport.cacheExtent](https://api.flutter.dev/flutter/rendering/RenderViewportBase/cacheExtent.html)