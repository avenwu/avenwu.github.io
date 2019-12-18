---
layout: post
title: "Flutter多平台适配机制就是这么简单"
description: "flutter适配多平台实现机制"
header_image: /assets/img/2019-12-18-01.jpg
keywords: ""
tags: [Flutter]
---
{% include JB/setup %}
![img](/assets/img/2019-12-18-01.jpg)

* 目录
{:toc #markdown-toc}

我们都知道到Flutter在表现层做到了多端一致性，通过Android、iOS各自平台下的渲染实现了一致的UI效果。
那么如果你只是要开发一个适配Android, iOS, Web的三方库，有什么好的简单思路？

## Flutter网络请求
在开发Flutter的时候可以使用[http核心库](https://pub.dev/packages/http)。也可以使用社区的其他封装类库，比如dio。两者的底层实现都是[http_parser](https://pub.dev/packages/http_parser)

如果开发者不小心在flutter中直接使用了平台相关的类库，则会导致扩平台运行出错，比如使用io包下的http在浏览器下执行肯定会报错。

[http核心库](https://pub.dev/packages/http)已经为我们做好了平台适配，下面看一下他是怎么做的适配：
```dart
import 'package:flutter/cupertino.dart';
import 'package:http/http.dart' as http;
void hello(){
  print('a.dart => hello');
  http.get('http://127.0.0.1:8080').then((response){
    debugPrint('response => ${response.statusCode} ${response.body}');
  });
}
```
这段代码可以跑在移动设备，也可以跑在浏览器设备，得到一致的输出效果。

## http核心库
现在我们以get请求为例，看一下他的内部逻辑：
![](/assets/images/http-platform-import.png)
在http接口类中，最终会执行`_withClient`来选用Client的实现类，类似静态代理效果。

具体来说，在编译为web使用时，最终导包使用的是`src/browser_client.dart`, 其底层实现是，`dart:html`下的`HttpRequest`, 最终用的是前端的ajax技术：`XMLHttpRequests`。
```
/// Used from conditional imports, matches the definition in `client_stub.dart`.
BaseClient createClient() => BrowserClient();

/// A `dart:html`-based HTTP client that runs in the browser and is backed by
/// XMLHttpRequests.
///
/// This client inherits some of the limitations of XMLHttpRequest. It ignores
/// the [BaseRequest.contentLength], [BaseRequest.persistentConnection],
/// [BaseRequest.followRedirects], and [BaseRequest.maxRedirects] fields. It is
/// also unable to stream requests or responses; a request will only be sent and
/// a response will only be returned once all the data is available.
class BrowserClient extends BaseClient 
```

针对非浏览器使用的是io类库，`src/io_client.dart`, 其底层实现是`dart:io`下的`HttpClient`

```dart
/// Used from conditional imports, matches the definition in `client_stub.dart`.
BaseClient createClient() => IOClient();

/// A `dart:io`-based HTTP client.
///
/// This is the default client when running on the command line.
class IOClient extends BaseClient 
```

## 条件导包
这里有个比较有意思的语法：
> `http`核心库是如何做到的的平台差异？

通过观察`src/client.dart`的导包情况，可以看到如下代码：
```dart
// ignore: uri_does_not_exist
import 'client_stub.dart'
    // ignore: uri_does_not_exist
    if (dart.library.html) 'browser_client.dart'
    // ignore: uri_does_not_exist
    if (dart.library.io) 'io_client.dart';
```
这里实际上使用的dart中的特殊语法:[条件导包](https://dart.dev/guides/libraries/create-library-packages#conditionally-importing-and-exporting-library-files)。
相关详情可以查阅dart文档。

> 简单来说就是利用有条件的import/export，在编译期间，差异化导包，从而可以实现平台适配。

使用条件导包的具体做法如下：
* 首先定义一个接口，用于多端实现；
* 接口类中利用import/export按需导入，导出对应的实现类库

```dart
export 'src/hw_none.dart' // Stub implementation
    if (dart.library.io) 'src/hw_io.dart' // dart:io implementation
    if (dart.library.html) 'src/hw_html.dart'; // dart:html implementation
```
## 运用场景
利用该机制可以方便的进行多平台适配。类似的dio也有一段导包差异逻辑`src/dio.dart`。
```dart
import 'entry_stub.dart'
// ignore: uri_does_not_exist
    if (dart.library.html) 'entry/dio_for_browser.dart'
// ignore: uri_does_not_exist
    if (dart.library.io) 'entry/dio_for_native.dart';
```

顺便看下dio和http的依赖情况。dio是一个http上传的封装库，提供了较多便捷的api，当然相对的也带了学习成本，具体是否采用就看项目的实际需要。
```
|-- dio 3.0.7
|   |-- http_parser 3.1.3
|   |   |-- charcode...
|   |   |-- collection...
|   |   |-- source_span...
|   |   |-- string_scanner...
|   |   '-- typed_data...
|   '-- path...

|-- http 0.12.0+2
|   |-- async...
|   |-- http_parser...
|   |-- path...
|   '-- pedantic...
```

## 参考
* https://dart.dev/guides/libraries/create-library-packages#conditionally-importing-and-exporting-library-files
* https://pub.dev/packages/http_parser
* https://pub.dev/packages/http


