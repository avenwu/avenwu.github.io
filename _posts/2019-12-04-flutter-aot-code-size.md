---
layout: post
title: "轻松掌握AOT产物大小"
description: "Dart AOT产物大小分析工具"
header_image: /assets/img/2019-12-04-01.jpg
keywords: ""
tags: [Flutter]
---
{% include JB/setup %}
![img](/assets/img/2019-12-04-01.jpg)

* 目录
{:toc #markdown-toc}

为了控制Flutter混合开发后，包体积不至于过大。我们需要了解Flutter构建App后的产物，其中与开发者关系密切的是Dart业务侧产生的机器码。也就是本文将分析的Dart AOT产物。

根据Flutter FAQ的解答[How big is the Flutter engine?](https://flutter.dev/docs/resources/faq#how-big-is-the-flutter-engine)，
> Android接入Flutter后，至少增加4.3MB，其中引擎3.2MB；
> iOS约10.9MB，部分原因是IPA的加密导致压缩率降低;

其中引擎占据大部分,随着业务模块组件增多，体积增加速度相对平缓。

了解AOT构建产物是为了掌握减小体积的方法。通过优化构建参数可以在不修改代码的情况下一般可以减少约8%的大小。当然配合代码优化还以在减少一些。

看一下引入机器码优化后的几份数据：

| 案例工程        | AOT产物减小百分比 |
| :---------------: | :-----------------: |
| flutter_gallery | 总大小减少~8.7%   |
| 闲鱼            | 总大小减少~8.7%   |
| 安居客"有料"    | app.so可以减小~8% |

*注：闲鱼数据来源于其对外分享数据，未做验证*

下面是笔者本机测试的有料数据：
```
7840264 bytes =》 7.48MB
7193096 bytes =》 6.86MB
```

本文标题为《轻松掌握AOT产物大小》，其实要掌握AOT产物是比较难的，不存在轻松的方法🙃。但既然你已经打开了文章，不妨继续了解下。

## Dart AOT产物
`Dart VM`支持JIT和AOT模式，在实际开发Flutter的过程中，开发环境下走的是JIT。我们通过release构建，可以生成dart的AOT代码。

这里没有指定Android/iOS平台，有需要也可以指定，默认会生成android-arm架构。

> flutter build aot --release

针对Android来说会生成app.so，他的目录在`build`下：
> flutter build aot --release --target-platform=android-arm64

```
├── aot
│   ├── app.dill
│   ├── app.so
│   ├── frontend_server.d
│   ├── gen_snapshot.d
│   └── kernel_compile.d
```
针对iOS来说会生成App.framework，

> flutter build aot --release --target-platform=ios --ios-arch=arm64

```
├── aot
│   ├── App.framework
│   │   └── App
│   ├── App.framework.dSYM.noindex
│   │   └── Contents
│   │       └── Resources
│   │           └── DWARF
│   │               └── App
│   ├── app.dill
│   ├── arm64
│   │   ├── App.framework
│   │   │   └── App
│   │   ├── App.framework.dSYM.noindex
│   │   │   └── Contents
│   │   │       ├── Info.plist
│   │   │       └── Resources
│   │   │           └── DWARF
│   │   │               └── App
│   │   ├── gen_snapshot.d
│   │   ├── snapshot_assembly.S
│   │   └── snapshot_assembly.o
│   ├── frontend_server.d
│   └── kernel_compile.d
```

当然构建ipa和apk的话还会有其他的产物，主要在flutter_assets目录中。

## AOT编译工具
最简单的编译模式下，我们没有干预AOT的编译参数，通过为`gen_snapshot`工具附加参数，可以实现更精细的配置和输出。

gen_snapshot所支持的参数，目前好像没有办法通过flutter help进行输出，最硬的方法是去分析编译流程的源码。
通过追加`--extra-gen-snapshot-options`参数,并为他附上具体的二级参数，可以实现很多配置，比如

> flutter build aot --release --extra-gen-snapshot-options="--dwarf_stack_traces,--print-snapshot-sizes,--print_instructions_sizes_to=build/aot.json"

关于gen_snapshot的流程，读者可以自行查阅文档, 也有不少分析的文章，如：http://gityuan.com/2019/09/21/flutter_gen_snapshot/

## 大小数据分析
在[Flutter瘦身大作战](https://www.yuque.com/xytech/flutter/hnxs1g)一文中，闲鱼开发者分享了他们对包大小的探索。从中我们可以知道，为了了解包大小激增原因，闲鱼团队在github上向Flutter团队提起了多个issues咨询。
通读完整个沟通流程后，我们可以得到一些有价值的信息：flutter开发团队提供了一些用于检测aot代码的工具。

到本文撰写的时候,可以配置的大小参数有以下几个，都是配置gen_snapshot的构建参数：

* --print-snapshot-sizes
* --print-instructions-sizes-to 输出aot生成代码的大小
* --write-v8-snapshot-profile-to 输出aot代码中所有类的v8快照格式文件

产出的数据都是json格式，可以利用dart脚本和Chrome可视化分析。

### "快照"大小
最简练的大小数据，是输出机器码产物中不同成分大大小情况：
![](/assets/images/print-snapshot-sizes.png)

其中的几个成分，也没有找到比较详细的说明，简单了解下大概含义

* 生成的代码 Instructions are the generated code. 
* 机器码的元数据 ReadOnlyData are generated code metadata (PcDescriptor, StackMap, CodeSourceMap) and strings
* 其他剩余对象，比如常量，VM的元数据如类，方法 Isolate/VMIsolate are the rest of objects. (e.g. constants you have written in your code and any other VM specific metadata like Class, Function etc objects).

### "指令"大小
除了按快归档的大小信息，我们还可以想办法得到dart编译后，不同代码指令的大小信息。
通过指定`--print-instructions-sizes-to`输出的机器码报告，并制定输出的文件名，他的格式是json.

![](/assets/images/aot-code-size.gif)

这个报告可以快速展示哪些代码块的体积大，比如由于as语法导致的自动生成异常检测和抛出的代码。原本几个字符的代码，瞬间变成几十字符代码
比如 `x as String` 会生成类似下面一段代码，因此大量使用as语法将会成倍增长代码量。
```
if (x.classId < A && x.classId > B) throw "x is not subtype of String";
```
根据开发者透露的信息，目前已经对代码生成量做了优化：https://dart-review.googlesource.com/c/78748

### v8格式快照信息
每错，这个v8和Chrome的V8是同一个单词。输出aot代码中所有类的v8快照格式文件，格式也是json，但是为了Chrome能够打开，可以命名为.heapsnapshot后缀，对数据格式感兴趣的同学可以研究下其字段的含义，这里我们直接用GUI查看了。

![](/assets/images/aot-code-object-level-size.png)

这个功能说实话，我到现在也没搞明白怎么快速上手，虽然是可视化显示了所以对象级别的大小数据，但是似乎没有问题检测功能。因此在没有具体可使用的场景下，也只能作为一份数据看一下了。

### 去除堆栈跟踪符号&混淆
既然是生成机器码，那就存在类似strip的概念，当然还有各种语言都有的代码混淆能力。

* --dwarf_stack_trace 表示在生成的动态库文件中，不使用堆栈跟踪符号
* --obfuscate 表示混淆，通过减少变量名/方法名的方式减小代码体积


## Dart VM工具

Dart SDK本身也提供了一些工具，具体在[pkg/vm/bin/](https://github.com/dart-lang/sdk/pkg/vm/bin/)目录下；

> pkg/vm/bin/compare_sizes.dart 

对比多份aot的代码大小

> pkg/vm/bin/run_binary_size_analysis.dart

将--print-instructions-sizes-to生成的json数据转导出Html的可视化报告

提醒一下，Flutter SDK虽然会自动下载Dart SDK到bin/cache目录，但是并不包本文问题提到的VM工具，需要的话要主动点吧，去github上clone或者单独剥出Dart脚本出来，千万不要去Flutter目录下生找了，你找不到的😅。

友情提供一份剥离好的脚本，下载后解压到工程的测试目录`test`下，然后就可以利用一些命令生成可视化的HTML报告了。
> 传送地址 =》[drart-vm-tools.tar.gz](/assets/files/drart-vm-tools.tar.gz)

![](/assets/images/run-dart-size.png)

## 小结
瘦身是一件不容易的事情，很多时候苍蝇再小也是肉。

## 参考
* https://www.yuque.com/xytech/flutter/hnxs1g
* https://github.com/flutter/flutter/issues/21813
* https://github.com/flutter/flutter/issues/18693
* https://github.com/dart-lang/sdk/blob/master/runtime/docs/aot_binary_size_analysis.md
