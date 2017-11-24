---
layout: post
title: "HLS点播服务搭建：gohls踩坑"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2017-11-24-03.jpg
keywords: ""
tags: []
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2017-11-24-03.jpg)

## 背景

在一个网红+直播的大环境中，不了解点直播技术怎么行呢？
正好组内也分享过一些直播的技术介绍，本文以此为契机，记录在实际搭建HLS后端遇到的问题和解决方法。

## HLS
直播或者点播主要依赖于流媒体协议，比如RTMP、HLS.
RTMP协议已经有很多年头了，HLS则是苹果公司推出的基于HTTP的流媒体协议。

简单来说HLS就是以小片段视频的形式提供给接收方，以此形成一种伪实时流的效果。
相关协议网上资料也很多，不在赘述。

## gohls

在实际后端搭建中，尝试过`nginx-rtmp-module`,`srs`,'gohls'三个开源的项目，但只有`gohls`最后成功了，不得不说，整个流程还是没有想象中容易的。

gohls本身并没有实现太多功能，正如其首页介绍，他的功能就是支持将一个目录下的视频一列表的形式提供出来，我们可以进行点播：
```
Simple server that exposes a directory for video streaming via HTTP Live Streaming (HLS). Uses ffmpeg for transcoding.

This project is cobbled together from all kinds of code I has lying around so it's pretty crappy all around. It also has some serious shortcomings.
```

虽然功能不是很多，但是从了解技术的角度来说，也是不错的选择。

## docker环境搭建

首先还是要配好一个用于实验的docker镜像环境，因为整个过程我们会涉及很多软件包的安装，非常不合适直接在本机中操作，采用docker就不用担心，用完就可以丢弃，不用担心安装了太多软件。

由于gohls主要是用go语言实现的一个简易server端，因此首先选一个golang的镜像，这里我们直接用了golang:1.8

```
docker run --rm --volume=$PWD:/srv/go -it -p 127.0.0.1:6000:8080 golang:1.8 /bin/sh
```
为了方便编辑代码，我们制定磁盘映射关系，同时制定端口映射，方便后期我们从本机请求docker内的服务。

## gohls安装

接下来克隆gohls源码：

```
git@github.com:shimberger/gohls.git
```
项目本身提供了三种预编译好的安装包，

* Windows (64 bit)
* Linux (64 bit)
* macOS (64 bit)

不过没有笔者使用的i386的安装包，因此我们需要已开发模式来运行gohls。

视频编码用的是ffmpeg，因此现将其下载到本地，可以新建一个tools目录，当然名字你可以任意，只要自己方便就行, 下载完后解压。

```
ffmpeg-release-64bit-static.tar.xz
```

接下来我们在镜像内做一下环境配置

```shell
# ffmpeg 
export PATH=/srv/go/gohls/tools:$PATH
echo $PATH
```

另外就是针对go的一些依赖包的安装，如果你直接根据项目的说明执行`run_dev`百分之百是要跪的。

下载go的核心库需要翻墙，如果不方便翻墙，那么需要手工从gitlab镜像下载。

```
# dependency
go get -u github.com/Sirupsen/logrus
go get -u github.com/google/subcommands
go get -u github.com/shimberger/gohls/hls
go get -u github.com/jteeuwen/go-bindata/...
```

这里总结了会遇到的几个需要安装的三方库和核心库。

```
# golang.org mirro
mkdir /go/src/golang.org
ln -s /go/src/github.com/golang /go/src/golang.org/x

# golang lib
go get -u github.com/golang/crypto/ssh/terminal
go get -u github.com/golang/sys/unix
```

`crypto/ssh/terminal`和`sys/unix`需要我们从github下载，然后指向到golang.org目录下
。你也可以直接进行文件拷贝，但是通过软连接更好一点，一次创建可以保证后续新下载的库同时生效。

## 运行gohls

现在我们已经安装完大部分环境和依赖，可以尝试运行一把，`./scripts/run_dev.sh`
不出以外的话，应该会报下面的警告提示，这是因为项目本身写的模糊不清。

```
Usage: bindata <flags> <subcommand> <subcommand args>

Subcommands:
	clear            Clears all caches and temporary files
	commands         list all command names
	flags            describe all known top-level flags
	help             describe subcommands and their syntax
	serve            Serves the directory for streaming


Use "bindata flags" for a list of top-level flags
exit status 2

```

运行gohls需要制定参数,如`serve videos`：

```
# ./scripts/run_dev.sh serve videos  
Path to ffmpeg executable: /srv/go/gohls/tools/ffmpeg
Path to ffprobe executable: /srv/go/gohls/tools/ffprobe
Home directory: /root/.gohls/
Serving videos in videos
Visit http://localhost:8080/
```

如果看到这里，基本说明服务端已经跑起来了。

我们试一下效果：

```
aven-mac-pro-2:gohls aven$ curl -i 127.0.0.1:6000
HTTP/1.1 302 Found
Location: /ui/
Date: Fri, 24 Nov 2017 07:48:13 GMT
Content-Length: 27
Content-Type: text/html; charset=utf-8

<a href="/ui/">Found</a>.
```

用chrome打开也不行，经过排查，发现端口6000有些问题，重新指定为4000, 并且还需要安装gulp，将ui下面的前端工程编译一下

```
npm install gulp-cli -g
npm install gulp -D
```
安装完毕后执行`gulp`

```
[14:43:53] Using gulpfile ~/hls/gohls/ui/src/gulpfile.js
[14:43:53] Starting 'sass'...
[14:43:53] Finished 'sass' after 10 ms
[14:43:53] Starting 'babel'...
[14:43:53] Finished 'babel' after 2.56 ms
[14:43:53] Starting 'img'...
[14:43:53] Finished 'img' after 748 μs
[14:43:53] Starting 'html'...
[14:43:53] Finished 'html' after 646 μs
[14:43:53] Starting 'vendor:css'...
[14:43:53] Starting 'vendor:js'...
[14:43:53] Starting 'fonts'...
[14:43:53] Finished 'fonts' after 746 μs
[14:43:53] Finished 'vendor:css' after 244 ms
[14:43:53] Finished 'vendor:js' after 259 ms
[14:43:53] Starting 'vendor'...
[14:43:53] Finished 'vendor' after 4.4 μs
[14:43:53] Starting 'default'...
[14:43:53] Finished 'default' after 3.72 μs
```
下载前端工程已经编译到ui/build目录下，

此时不需要用node来启动什么服务，因为后端是golang实现的，

可以在`scripts/run_dev.sh`看到下面的内容：

```
#!/bin/bash

PATH=$GOPATH/bin/:$PATH
cp buildinfo.go.in buildinfo.go
go-bindata -debug -prefix ui/build ui/build/...
DEBUG=true go run *.go ${@:1}
```

最后我们在此验证一下

```
ven-mac-pro-2:gohls aven$ curl -i http://127.0.0.1:4000/ui/
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Date: Fri, 24 Nov 2017 07:48:45 GMT
Content-Length: 486

<!doctype html>
<html lang="en">
<head>
	<meta charset="utf-8">
	<meta http-equiv="X-UA-Compatible" content="IE=edge">
	<meta name="viewport" content="width=device-width, initial-scale=1">

	<title>Videos</title>

	<link rel="stylesheet" href="/ui/assets/css/vendor.css"></script>
	<link rel="stylesheet" href="/ui/assets/css/app.css">
</head>
<body>
	<div id="app">

	</div>
	<script src="/ui/assets/js/vendor.js"></script>
	<script src="/ui/assets/js/app.js"></script>
</body>
</html>
```

![2017-11-24-01.png](http://7u2jir.com1.z0.glb.clouddn.com/img/2017-11-24-01.png)

点击列表可以播放视频，通过后端也可以查看到日志，视频是以分片形式不断通过http协议传输。

![2017-11-24-02.png](http://7u2jir.com1.z0.glb.clouddn.com/img/2017-11-24-02.png)

## 小结

gohls的介绍到此基本结束，在搭建服务过程中遇到很多错误，也走了不少弯路。
