---
layout: post
title: "Flutter Web实现目录选择探究"
description: "Flutter Web实现目录选择"
header_image: /assets/img/2020-02-11-01.jpg
keywords: "目录选择"
tags: [Ffutter]
---
{% include JB/setup %}
![img](/assets/img/2020-02-11-01.jpg)

* 目录
{:toc #markdown-toc}

在日常使用浏览器的时候，上传文件是很常见的操作，一般都是弹出系统的文件选择器，用户勾选某个文件，确认后即可上传；下载也类似，都是打开文件选择器，选择到合适位置后，命名保存；

> 那如果我们需要打开文件选择器，并且选择选择目录怎么实现呢？

要实现这个效果，在H5中我们可以使用`input`标签，结合`webkitdirectory`属性

```
<input type="file" id="ctrl" webkitdirectory directory/>
```
这里我们主要通过了非标准的属性`webkitdirectory`实现了目录选择能力。

上传目录是比较危险的事情，他会递归查找目录的所有子文件，目前在chrome中，选择后会给出二次警告确认。

![](/assets/images/directory-alert.png)

另外还有一个已知的bug
> https://bug1326031.bmoattachments.org/attachment.cgi?id=8822154

## Flutter Web目录选择
但是在Flutter-Web开发中，input file对应的API如下，它并不支持webkitdirectory属性：
```dart
/**
 * A control for picking files from the user's computer.
 */
abstract class FileUploadInputElement implements InputElementBase {
  factory FileUploadInputElement() => new InputElement(type: 'file');

  String accept;

  bool multiple;

  bool required;

  List<File> files;
}
```

根据H5的实现方案，我们知道只要想办法支持`webkitdirectory`配置，理论上就可以做到和H5一样的效果；

## 自定义Inupt标签类
阅读input的源码可知，针对不同type，Flutter实现了不同的InputElement子类。

如果尝试继承该类，会由于基类的定义导致属性扩展失效：

```dart
@Native("HTMLInputElement")
class InputElement extends HtmlElement
    implements
        HiddenInputElement,
        SearchInputElement,
        TextInputElement,
        UrlInputElement,
        TelephoneInputElement,
        EmailInputElement,
        PasswordInputElement,
        DateInputElement,
        MonthInputElement,
        WeekInputElement,
        TimeInputElement,
        LocalDateTimeInputElement,
        NumberInputElement,
        RangeInputElement,
        CheckboxInputElement,
        RadioButtonInputElement,
        FileUploadInputElement,
        SubmitButtonInputElement,
        ImageButtonInputElement,
        ResetButtonInputElement,
        ButtonInputElement {
  factory InputElement({String type}) {
    InputElement e = document.createElement("input");
    if (type != null) {
      try {
        // IE throws an exception for unknown types.
        e.type = type;
      } catch (_) {}
    }
    return e;
  }
```
因此通过自定义InputElement来增加可控参数是行不通的。

## extension 扩展
Dart 2.6以后新增了extension语法，可以对已存在的框架类进行扩展。

```dart
extension InputExtension on html.FileUploadInputElement {
  bool get webkitdirectory {
    return true;
  }

  bool get directory {
    return true;
  }
}
```
通过验证，发现虽然增加了扩展get属性，但是仍然没有效果。

## 构造Element
在往下一层分析就是html的Element构建，很幸运，我们看到了一个`Element.html`的api。
根据注释，该方法仅支持单标签，所以这样写是可以的：
```dart
var element = new Element.html('<div class="foo">content</div>');
```

类似的我们就input标签传入验证效果：
```dart
html.Element.html('<input type="file" id="ctrl" webkitdirectory directory multiple/>');
```
刚输入就IDE便提示警告，警告信息其实就是前面提到过的，非标准属性问题，这个我们可以跳过检测：
> '<!--suppress HtmlUnknownAttribute --><input type="file" webkitdirectory directory/>'

运行后发现没有效果，同时注意到console内输出了以下信息：

>Debug service listening on ws://127.0.0.1:53348/dStY_H3d3dg=
>Removing disallowed attribute <INPUT directory="">
>Removing disallowed attribute <INPUT webkitdirectory="">

很直白，原理Flutter内部处理黑名单，还有白名单，所有支持的html及其属性，都会被加入白名单，如果属性不在白名单内，在构造的时候就会自动擦除不支持的属性。

> 这个能力简直惨无人道😂

继续分析源码，我们追踪到了个能力的对应逻辑代码；通过validator检测器，直接将不符合的属性予以剔除。

```dart
// TODO(blois): Need to be able to get all attributes, irrespective of
// XMLNS.
var keys = attrs.keys.toList();
for (var i = attrs.length - 1; i >= 0; --i) {
  var name = keys[i];
  if (!validator.allowsAttribute(
      element, name.toLowerCase(), attrs[name])) {
    window.console.warn('Removing disallowed attribute '
        '<$tag $name="${attrs[name]}">');
    attrs.remove(name);
  }
}

```
如果想要绕过去，那是不可能的，因为构造器和属性集合是cosnt类型，不能修改🤣。

这个白名单定义如下：
```dart
static const _standardAttributes = const <String>[...];
```

其中input标签支持的属性有这些，形式均为`标签名大写::属性小写`，可以看到确实不支持目录属性，毕竟它不是w3c的标准，也能理解：

```
'INPUT::accept',
'INPUT::accesskey',
'INPUT::align',
'INPUT::alt',
'INPUT::autocomplete',
'INPUT::autofocus',
'INPUT::checked',
'INPUT::disabled',
'INPUT::inputmode',
'INPUT::ismap',
'INPUT::list',
'INPUT::max',
'INPUT::maxlength',
'INPUT::min',
'INPUT::multiple',
'INPUT::name',
'INPUT::placeholder',
'INPUT::readonly',
'INPUT::required',
'INPUT::size',
'INPUT::step',
'INPUT::tabindex',
'INPUT::type',
'INPUT::usemap',
'INPUT::value',
```

## 自定义H5校验器
既然默认情况下会命中白名单，那我们顺藤摸瓜，看看封装了白名单逻辑的H5校验器是否可以替换；

```dart
factory Element.html(String html,
  {NodeValidator validator, NodeTreeSanitizer treeSanitizer}) {
var fragment = document.body.createFragment(html,
    validator: validator, treeSanitizer: treeSanitizer);

return fragment.nodes.where((e) => e is Element).single;
}
```
注意这里有一个可选参数`NodeValidator`,通过上下文，可知他的默认实现和我们发现的H5白名单是有关联的：
```
if (validator == null) {
  if (_defaultValidator == null) {
    _defaultValidator = new NodeValidatorBuilder.common();
  }
  validator = _defaultValidator;
}
```

common方法会添加默认的校验规则：
```dart
allowHtml5();
allowTemplating();
```

所以我们的解决办法就是提供个性化的校验器，让input标签支持`webkitdirectory`
```dart
var input = html.Element.html('<input type="file" webkitdirectory directory/>',
  validator: html.NodeValidatorBuilder()
    ..allowElement('input', attributes: ['webkitdirectory', 'directory'])
    ..allowHtml5()
);
```

注意这里的顺序不可写反，如果先添加allowHtml5，则自定义的校验会失效。

到这里我们已经构造了一个虚拟的input标签，通过触发点击事件即可以模拟出目录选则的效果。


## W3C协议规范

在整个源码分析和协议查找过程中，找到了一些针对目录的协议。

### File and Directory Entries API
这个协议处于Draft状态，但是Chrome高版本已经支持；

和本文相关的细节是，里面定义了`webkitEntries`。
https://wicg.github.io/entries-api/#dom-file-webkitrelativepath

```javascript
document.querySelector('#a').addEventListener('change', e => {
  for (const entry of e.target.webkitEntries)
    handleEntry(entry);
  }
);
```

webkitEntries是一个`FileSystemEntry`的数组，理论上可以拿到一个fullPath绝对路径，但是通过实践发现了两个bug；
* 只能通过drag and drop的拖拽操作，整个数组才有内容；
* 即使有内容，fullPath的所谓全路径，也不是我们常规理解的绝对路径，虽然他也是以斜杠起头`\`

```javascript
interface FileSystemEntry {
    readonly attribute boolean isFile;
    readonly attribute boolean isDirectory;
    readonly attribute USVString name;
    readonly attribute USVString fullPath;
    readonly attribute FileSystem filesystem;

    void getParent(optional FileSystemEntryCallback successCallback,
                   optional ErrorCallback errorCallback);
};
```
所以通过这个API无法实现本机目录的获取。
完整的input标准协议可以在这里查阅：[The input element](https://html.spec.whatwg.org/multipage/input.html#the-input-element), 它是正式标准，不包括其他草案。

查看某个API的支持情况（不管是正式还是草案中的api），都可以在这里查阅[https://caniuse.com/](https://caniuse.com/#feat=input-file-directory)

查询html规范标准后，发现标准中并没有定义`webkitdirectory`，也就是说行业标准中其实并没有统一的选择目录的api。但是截至本文编写时，目前PC浏览器基本都支持了该属性。移动浏览器基本没有支持。

## 思考
在现有的H5标准中并没有现成的，选择目录并获取绝对路径的解决方案。反过来我们也可以继续思考下：
> web应用本身是用来和服务端交互的，服务端也无法访问客户端的绝对路径。只有部署在本机的服务端正好可以获取本机路径。从这个角度来说，不支持目录绝对路径获取也是可以理解的。

只有把web部署在本机运行时，读写PC的目录才是可行的。这里我们正好是部署在本机的前端工程，因此可以通过后端服务提供系统目录的能力，前端作为系统目录的展示层，对路径的选择和查找通过跨进程通信解决，比如通过HTTP，WebSocket通信也可以解决。

这种方案需要开发者自己定义并实现一个基本的选其效果，可以参考系统选择器，下面是我们实现的一个效果。
![](/assets/images/directory-picker.png)