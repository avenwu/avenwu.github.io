---
layout: post
title: "Flutter CG技术运用"
description: "Flutter Code Generation代码生成"
header_image: /assets/img/2019-11-03-01.jpeg
keywords: ""
tags: [Flutter]
---
{% include JB/setup %}
![img](/assets/img/2019-11-03-01.jpeg)

* 目录
{:toc #markdown-toc}

本文介绍Flutter中的Code Generation技术，

1. 如何配置应用代码生成能力的插件
2. 自定义代码生成插件

# **插件配置**

pubspec.yaml内配置buidlers依赖

```
builders:
  json_widget: any
```

CG错误情况：pubspec.yaml not found, 检查本地Flutter环境，确保在稳定stable分支，master可能会编不过

代码生成位置 `.dart_tool/build/generated/project_name/lib`

json_widget样例效果

​             ![img](https://qqadapt.qpic.cn/txdocpic/0/1087fe1d8f0c61bf66b4d391d588cbf6/0)             


# **自定义插件**

创建插件类，需要继承package:build/build.dart下的Builder接口，并实现相应的函数

- buildExtensions  定义输出文件的后缀名，形式为键值对 ，输入后缀：输出后缀数组
- build 定义构建函数


示例如下，创建的一个插件类，对外的入口函数：

- 返回类型为Builder，入参为Builderoptions。 
- 该入口函数名需要配置在build.yaml中

```dart
import 'package:build/build.dart';

/// The builder factory used by the `build.yaml` script.
Builder simpleBuilder(BuilderOptions options) => SimpleBuilder();

/// A trivial builder which copies the contents of a `spec` file into a `dart` file.class 
SimpleBuilder extends Builder {
  @overrideMap<String, List<String>> get buildExtensions => const <String, List<String>>{'.spec' : <String>['.dart']};

  @overrideFuture<void> build(BuildStep buildStep) async {
    // The asset id identifiesfinal AssetId output = buildStep.inputId.changeExtension('.dart');
    final String contents = await buildStep.readAsString(buildStep.inputId);
    buildStep.writeAsString(output, contents);
  }
}
```

只有入口函数所在文件需要在工程根目录下，一般为project/lib/xxx

```yaml
# Read about `build.yaml` at https://pub.dev/packages/build_configbuilders:
  simple:
    import: "package:simple_codegen/builders.dart"
    builder_factories:
      - simpleBuilder
    build_extensions: {'.spec':['.dart']}
    auto_apply: all_packages
```

# **通用技术点**

在实现build过程中，经常使用的一些操作

## **修改输出后缀名**

使用changeExtension函数

```dart
final AssetId outputId = buildStep.inputId.changeExtension('.dart');
```

## **保存文件**

结合获取的输出标识AssetsId，利用writeAsString函数

```dart
buildStep.writeAsString(outputId, outputString);
```

## **读取文件内容**

使用readAsString函数

```dart
source = json.decode(await buildStep.readAsString(buildStep.inputId));
```

## **格式化代码**

使用DartFormatter工具类

```dart
outputString = DartFormatter().format(output.toString()).toString();
```

**参考**

1. https://github.com/flutter/flutter/wiki/Code-generation-in-Flutter
2. https://github.com/jonahwilliams/json_widget                                                                                                                                                                                     