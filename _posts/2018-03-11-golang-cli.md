---
layout: post
title: "Golang CLI"
description: "通过Golang编写CLI工具，并通过Homebrew发布共享"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2018-03-11-01.png
keywords: "Golang CLI, Homebrew"
tags: [Go]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2018-03-11-01.png)

本文介绍如何通过Golang开发自己的脚本工具，并共享给多人使用。

## 0x01 初识Golang

首先我们通过三个问题来简单认识下Golang：

1. Golang是什么？
2. 发展现状怎么样？
3. Gopher都在使用哪些开发工具？

> 1）Golang是什么？

简单来说Golang是一门语言，由Google公司设计并主导的一门开源的语言，主要用于后端开发。

Golang一般也叫做Go，使用Golang的开发者一般自称为Gopher。其标识是一个类似`土拨鼠`的形象，至于到底是什么鼠，原谅我也不认识这种生物。

![Golang Logo](http://7u2jir.com1.z0.glb.clouddn.com/img/QQ20180311-150942@2x.png)

为什么会出现这门语言? 

难道大公司喜欢推新的语言？当然这是玩笑话。Golang的出现，是为了解决特定的问题，对现有语言及解决方案的不够满意，以及整合各种优秀特性，最终产生了这门语言。在编程语言中中既有C/C++，也有Java，Python，PHP，JavaScript等，从语言的执行速度来看，Golang比C/C++略慢，但是比其他语言都要快。话说回来，速度问题其实并不在本文讨论范围之内。

在官网[https://golang.org](https://golang.org/) 上，有一段是这么说的：

```
Go is an open source programming language that makes it easy to build simple, reliable, and efficient software.
```

几个关键词：`开源`，`简单`，`可靠` ，`高效` 



> 2）Golang发展现状怎么？

Golang组织了几次开发者问卷调查，感兴趣的话，可以去看看，调查维度非常广，挺有意思的，2017问卷报告：：[Go 2017 Survey Results](https://blog.golang.org/survey2017-results)

**整体来说Golang越来越收到欢迎，并运用于工作当中（69%）**

![Golang Work](http://7u2jir.com1.z0.glb.clouddn.com/img/QQ20180309-173219@2x.png)

**使用Golang开发什么项目，主要是后端服务，CLI工具**

![Golang Product](http://7u2jir.com1.z0.glb.clouddn.com/img/QQ20180309-173312@2x.png)



> 3）Gopher都在使用哪些开发工具？

使用Golang的用户大多数是*nix用户，工具选用已VSCode和Vim为主。下面两个数据表来源于2017年度的开发者问卷调查。

![Golang Enviroment](http://7u2jir.com1.z0.glb.clouddn.com/img/QQ20180309-173357@2x.png)

![Golang IDE](http://7u2jir.com1.z0.glb.clouddn.com/img/QQ20180309-173407@2x.png)

## 0x02 CLI开发

终于到了介绍语言本身的时候，这一节我们通过两个例子来入门Golang。

### 案例：Hello World

来个Hello World吧，下面是从Golang官网截取的示例：

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, 世界")
}
```

执行结果如下：

```shell
Hello, 世界

Program exited.
```

在这个例子中，涉及包名，导包，程序入口函数，输出语句，字符串之类。具体语法可以参考[Golang API文档](https://golang.org/doc/)

### 案例：清理脚本

`Hello Wrold`很简单对不对？几乎所有的语言的hello world都很简单，因为只有一个输出的知识点。现在我们来开发一个小小的工具。

首先我们介绍一下需求背景：

> 1. 在使用IDE的时候经常会配置很多源码工程，被关联的源码可以通过IDE的clean或者脚本来删除一些构建的临时/编译文件，比如`build`目录，`gen`目录等等；
> 2. 现在我们希望写一个这样功能的脚本，它能够根据指定目录删除临时/编译文件，以节省磁盘空间；

在开发之前，我们定义脚本的使用形式，比如需要哪些参数（后续可以根据需要调整）

**脚本名**：`deleteBuild`

**参数**：

>
1. **-d** 调试模式，不直接删除，进输出待删除文件信息  
2. **-n** 需要删除的文件目录，比如`build`，或者`gen`  
3. **-p** 查找路径，比如当前目录`./`

脚本使用说明，在执行脚本命令的时候，可以打印出出使用说明如下：

```shell
aven-mac-pro-2: aven$ deleteBuild 
Delete build files recursively
Usage: deleteBuild [options]
Options:
  -d, --debug
    	Skip delele for debug
  -n, --name string
    	specific file name needs to be delete (default "build")
  -p, --path string
    	directory to start with (default "./")
```

脚本测试，通过**-d**参数，我们查看一下当前目录下递归删除所有build目录/文件的情况：

```shell
aven-mac-pro-2: aven$ deleteBuild -d 
Found client/build
Found node_modules/core-js/build
Found node_modules/fbjs/node_modules/core-js/build
Found node_modules/fsevents/build
Found node_modules/mime/build
Found node_modules/recompose/build
Found node_modules/webpack-core/node_modules/source-map/build
Delete client/build
Delete node_modules/core-js/build
Delete node_modules/fbjs/node_modules/core-js/build
Delete node_modules/fsevents/build
Delete node_modules/mime/build
Delete node_modules/recompose/build
Delete node_modules/webpack-core/node_modules/source-map/build
```

### 编码：deleteBuild脚本实现

脚本功能和参数定义我们已经明确了，现在思考一下如何实现这个脚本。

> 1. 参数接收，处理
> 2. 遍历目录
> 3. 匹配文件名
> 4. 执行删除操作/日志输出

根据这个思路，可以用任何你所熟悉的语言进行翻译并实现。我们这里采用Golang。核心代码如下：

```go
if _, err := os.Stat(rootPath); err != nil {
	die("directory does not exist", err)
}
fileList := []string{}
filepath.Walk(rootPath, func(path string, info os.FileInfo, err error)error {
	if info.Name() == folderName {
		color.Green("Found %s", path)
		fileList = append(fileList, path)
	}
	return err
})
for _, file := range fileList {
	color.Yellow("Delete %s", file)
	if debug {
		continue
	}
	if e := os.RemoveAll(file); e != nil {
		color.Red("Remove failed %v", e)
	}
}
```

这里有一个可以考虑优化的点是，遍历目录的层级控制，上述实现中通过`Walk`方法来遍历文件，代码书写比较简洁，但是可控的部分就比较少。

如果是`debug`的情况，我们跳过删除操作。

完整代码详见：[main.go](https://github.com/hacktons/homebrew-cli/blob/master/deleteBuild/main.go)

代码写好了肯定就是运行了，运行go代码比较简单，我们这里只有一个源代码，因此直接执行`go run main.go`即可。

到这里，我们已经设计并实现了整个清理脚本，在下一节我们介绍如何发布脚本并共享给其他人使用。

## 0x03 工具发布 

作为mac党，应该没有不知道`Homebrew`的吧，如果你确实不知道，那么对不起耽误你时间了，这篇文章也不用继续看下去了:)

言归正传，`Homebrew`是专为mac下安装软件开发的一个安装包管理软件，使用`Ruby`开发而成。

> **PS**: 如果使用其他平台如`linux`和`windows`，也可以输出响应平台下的可执行文件，如exe。

### Homebrew

>  所以我们开发的Golang脚本是不是可以通过`Homebrew`来发布/安装?

当然是可以的。`Homebrew`的核心库要求所有提交的软件必须是以源代码形式下载并编译安装，因此我们可以根据要求编写`Formula` ，如果要合并官方brew的仓库，需要遵守以下规则，如下：

> https://docs.brew.sh/Acceptable-Formulae

### 维护Tap

这里我们并不打算走官方的合并仓库，因为我不希望用户在使用我们的脚本时还需要Golang的编译环境，这是不能接受的。因此我们打算本地编译好可执行文件，然后提交到自定义的仓库。

[https://docs.brew.sh/How-to-Create-and-Maintain-a-Tap](https://docs.brew.sh/How-to-Create-and-Maintain-a-Tap)

建立自己的软件仓库也需要符合命名规则，这样可以使我们的操作更方便。仓库名以homebrew开头，形如`homebrew-xxx`，下面是我们的仓库：

[https://github.com/hacktons/homebrew-cli](https://github.com/hacktons/homebrew-cli)

安装的时候比核心库需要一个路径即你的github用户名和仓库名，比如hacktons/cli:

```shell
aven-mac-pro-2:Desktop aven$ brew install hacktons/cli/delete
Warning: hacktons/cli/delete 0.0.1 is already installed
```

### 可执行文件

现在我们需要准备发布用的可执行文件，目前来说基本都普及64系统了，因此我们仅针对64的macOS，Windows，Linux编译。

| 平台      | 可执行文件                                    | 位数   |
| ------- | ---------------------------------------- | ---- |
| macOS   | Mach-O 64-bit executable x86_64          | 64   |
| Windows | PE32+ executable (console) x86-64, for MS Windows | 64   |
| Linux   | ELF 64-bit LSB executable, x86-64        | 64   |

编译命令很简单:`go build` 会针对当前系统状态，构建出本机可运行的可执行文件。

如果要跨平台编译，也很简单，跟上系统参数：`GOOS=$os` 其中$os表示目标系统，同时也可以指定输出文件名。系统的对应值可以在Golang获取到 [https://golang.org/doc/install/source#environment](https://golang.org/doc/install/source#environment)。

| `$GOOS`     | `$GOARCH`  |
| ----------- | ---------- |
| `android`   | `arm`      |
| `darwin`    | `386`      |
| `darwin`    | `amd64`    |
| `darwin`    | `arm`      |
| `darwin`    | `arm64`    |
| `dragonfly` | `amd64`    |
| `freebsd`   | `386`      |
| `freebsd`   | `amd64`    |
| `freebsd`   | `arm`      |
| `linux`     | `386`      |
| `linux`     | `amd64`    |
| `linux`     | `arm`      |
| `linux`     | `arm64`    |
| `linux`     | `ppc64`    |
| `linux`     | `ppc64le`  |
| `linux`     | `mips`     |
| `linux`     | `mipsle`   |
| `linux`     | `mips64`   |
| `linux`     | `mips64le` |
| `linux`     | `s390x`    |
| `netbsd`    | `386`      |
| `netbsd`    | `amd64`    |
| `netbsd`    | `arm`      |
| `openbsd`   | `386`      |
| `openbsd`   | `amd64`    |
| `openbsd`   | `arm`      |
| `plan9`     | `386`      |
| `plan9`     | `amd64`    |
| `solaris`   | `amd64`    |
| `windows`   | `386`      |
| `windows`   | `amd64`    |

但是一般我们并不需要发这么多平台的目标文件，根据实际情况，我们写了一个脚本来简单处理，默认构建三个系统下的可执行文件。

```shell
#!/usr/bin/env bash
##################################################################
##
##  Simple script to build binnary files for:
##
##  1. macOS/darwin
##  2. Linux 
##  3. Windows
##  
##  You may ship specific os based binnary with predifined contant
##  https://golang.org/doc/install/source#environment
## 
##  Author: Chaobin Wu
##  Email : chaobinwu89@gmail.com
##
#################################################################

arg="os"
if [ $# == 1 ]; then
  script="${0##*/}"
  GOOS=$1 go build
else
  osArray=(darwin windows linux)
  dir=`pwd`
  fileName=${dir##*/}
  for os in ${osArray[@]}; do
    echo "build for $os"
    name=$fileName-$os
    if [ $os = "windows" ]; then
      name=$fileName-$os.exe
    fi
    GOOS=$os go build -o $name
    echo "build success: $name"
  done
fi
```

得到了可执行文件后，实际上你就可以分享给其他人使用了。

可以结合前面说的Homebrew进行发布，这里介绍一个专门用于简化发布Go脚本的工具。

> [https://goreleaser.com/](https://goreleaser.com/)

这个项目也是用Golang实现的，他的优点是将Golang的编译与GitHub仓库配合的非常好，可以直接实现每个版本release和tag处理等等，并且可以自动生成Formula之类的配置，只需要一些相关的yml配置。

最后看一个Formula的示例，也就是本文deleteBuilde配置：

```yaml
class Delete < Formula
  desc "Simple scripts that help to ease handy work daily, most of these cli tools was written in Golang"
  homepage "http://hacktons.cn/homebrew-cli"
  url "https://github.com/hacktons/homebrew-cli/releases/download/v0.0.1/deleteBuild_0.0.1_macOS_64-bit.tar.gz"
  sha256 "8de05ac044444d33beca2f27bc274939cdcf2cd1e5f00229c64bf2b1aa02f82a"

  version "0.0.1"

  def install
    bin.install "deleteBuild"
  end

  test do
    system "bin/deleteBuild"
  end
end

```



## 0x04 The End

如果你也有些小想法想，那就赶快行动吧。Happy Coding!!!