---
layout: post
title: "与七牛云的分分合合"
description: ""
header_image: /assets/images/2019-04-19-01.jpg
keywords: ""
tags: []
---
{% include JB/setup %}
![img](/assets/images/2019-04-19-01.jpg)

## 背景

18年的时候七牛回收了测试域名，导致笔者的外链全部失效。

七牛云是云服务提供商，国内的一家创业公司，很多年头了，现在仍然活跃着。笔者主要使用的七牛的云存储服务，了解更多信息可以访问[qiniu.com](https://portal.qiniu.com/create)

## 外链失效问题

在搭建一些web服务的过程中，积累了一些图片等物料资源，笔者将这些东西托管在七牛上面，提供给web服务使用，类似专用图床，提供国内访问更快的速度。

![七牛-存储](/assets/images/web-qiniu.png)

由于不可知的原因，七牛上的内容不能再通过系统自动分配的外链地址进行下载或访问，外链必须是自定义的的经过公安部备案的域名。

当然如果你在七牛回收域名之前已经把资源已经做了备份，那么问题不大，可以绑定域名继续使用七牛，也可以将资源放到其他服务器使用。
如果和笔者一样，根本没有注意到七牛的域名回收邮件，或者已经超过回收时间，那么按照官方的工单恢复，则必须绑定域名才可以找回服务器上的资源。

如果你没有备案过的域名，那么如果找回资源？

## 找回七牛资源

通过七牛后台，我们知道必须要绑定域名。

![七牛配置后台](/assets/images/qiniu-restore-1.png)

不绑定域名则，后台无法下载空间内的资料，经过测试通过qiniu的cli工具也不能拉取资源。

如此陷入两难境地：

* 公安备案，为了几个简单的博客和web服务，这操作有点恶心了，并且备案需要很多个人信息，堪比查水表
* 放弃资料，其实也没什么大不了，不过对已经存在的web服务非常不友好，会增加文章阅读障碍

### 借壳绑定
**最近偶然想到一个骚操作，是不是可以将现有的已备案的网站做绑定？**

随便找一个国内的大站点，比如BAT的首页都可以。直接添加，等五分钟后后台提示，成功了：）

绑定域名后，后台点击可以进行下载，不过会下载失败。原因是这些站点不是你的，仅仅绑定是不可以产生有效外链的。

### 批量下载
前面我们知道后台下载失败，是由于七牛走了域名外链，这些外链不合法，所以失败。如果使用过七牛的cli，会知道，这个工具是不通过外链下载的，
如果不配置，他将直接从源存储站点下载，这就给了我们`可乘之机`。

![七牛cli说明](/assets/images/qiniu-cli-tool-usage.png)

![七牛cli](/assets/images/cli-qniu.png)

```shell
qshell qdownload 10 qiniu-config.conf
```

![七牛同步记录](/assets/images/qiniu-download-log)

## url替换
得到了资源后，可以放到其他空间，或者直接放到使用的项目当中，作为源码的一部分，在这个过程中会涉及大量的替换动作，我们可以通过脚本来处理；

**比如替换所有md文档内的外链地址**

```shell
posts_md=(_posts/*.md)
for post in ${posts_md[@]}; do
	echo $post
	if [[ -f $post ]]; then
		sed -i "" 's/http:\/\/7u2jir\.com1\.z0\.glb\.clouddn\.com/\/assets/g' $post 
	fi
done
```

这个操作会产生很多git修改文件，接下来需要批量提交文件，由于笔者本地有很多不需要提交的文件，两种混合在一起导致手工commit变得不太方便。
同样，借助git的输出信息，我们可以精确的匹配文件，只提交修改的文件，不提交为其他本地文件。

![git-batch-add-modify](/assets/images/git-batch-add-modify.png)
 
```shell
# 批量添加已修改状态的文件（忽略未跟踪的文件）
git status -s|grep "^ M"|awk '{ print $2 }'|xargs git add
```

## 小结

到这里七牛的资源找回和替换，基本告一段落。由于没有使用国内站点存储图片，在访问效率上会打一定折扣，但比起没有图片总是好一点的。

## 参考
* [https://developer.qiniu.com/sdk#official-tool](https://developer.qiniu.com/sdk#official-tool)
* [https://developer.qiniu.com/kodo/tools/1302/qshell](https://developer.qiniu.com/kodo/tools/1302/qshell)
* [https://github.com/qiniu/qshell/blob/master/docs/qdownload.md](https://github.com/qiniu/qshell/blob/master/docs/qdownload.md)
