---
layout: post
title: "Jekyll Docker迁移笔记"
description: ""
header_image: /assets/img/2017-06-26-01.png
keywords: "jekyll docker"
tags: [docker]
---
{% include JB/setup %}
![img](/assets/img/2017-06-26-01.png)

# 前言

在很久以前，出于便捷性，用jekyll搭建了静态博客，并从博客园迁移至此；在使用的这几年中也陆续遇到过一些问题，大部分是由于jekyll版本兼容和依赖库变更，导致编译失败；
因此有了今天的迁移到Docker

# Docker

Docker其实已经火了挺久了，但是一直觉得用不着，所以也就没有过多了解，最近通过Docker的官方介绍，产生了利用Docker环境来解决jekyll的潜在编译问题；

我们的目标是，希望维持Jekyll的简洁性，希望环境问题再也不干扰博客的编写，从目前来看，采用Docker配置环境可以解决如下问题：

* Jekyll的2.x和3.x差异
* macOS的版本差异，比如默认ruby的安装目录和权限问题
* 使用环境统一

# Jekyll Docker

简单Google以下可以发现Jekyll官方也推出了[Jekyll Docker镜像](https://github.com/jekyll/docker)，那么直接按照操作指南依葫芦画瓢就可以了吧；

可惜现实比没有那么顺利，首先Jekyll 的说明非常少，而且不明确; 

默认提供了三个版本的镜像，分别是Standard，Builder以及Minimal，看了下解释，大概是依赖关系不同，标准版会附带很多依赖包，Minmal是精简版，那么是不是任意一个版本都可以呢？

我们试着运行了其中的minimal，指令如下：

{% highlight shell %}

export JEKYLL_VERSION=3.5
docker run --rm \
  --volume=$PWD:/srv/jekyll \
  -it jekyll/minimal:$JEKYLL_VERSION \
  jekyll build

{% endhighlight %}

实际结果并没有出现常规jekyll build之后的输出，所以感觉一定是哪里姿势不对，但是又找不到错在哪里。没办法只能从Docker 的入门开始，去了解如何启动一个docker进行，进而解决我们的启动本地站点。

# Docker 入门

直接去查阅Docker的官方说明即可[https://docs.docker.com/](https://docs.docker.com/), 注意虽然也有国内的一些翻译版本，等各种资料，笔者还是建议读者要慎重选择翻译版本，主要是考虑到更新问题，国内的翻译资料一般很难保证变更的正确性，比如关于Docker的安装，在官网其实就介绍的比较多，比如Docker Toolbox和 Docker for Mac等等。
以macOS来说，现在安装和使用有一些不同，只需要同个Docker for Mac下载一个dmg安装，启动，就可以直接在终端运行docker命令，不需要再套一层machine了，所以这也引出另一个问题，这个后续再说。

文档比较多，一章章看完入门的介绍大致差不多可以再次动手了。

我们先来理解一下Jekyll Docker提供的命令，其中各参数段的含义：

{% highlight shell %}

export JEKYLL_VERSION=3.5
docker run --rm \
  --volume=$PWD:/srv/jekyll \
  -it jekyll/minimal:$JEKYLL_VERSION \
  jekyll build

{% endhighlight %}

* export 这个不说也知道，是shell里面导出环境变量的，应为后面再指定版本时用的是这个方式，实际上，如果只直接执行命令，可以不用这么麻烦，如果是写成脚本的话导出确实更有可读性

* run 启动一个镜像
* --rm 和删除有点关系，表示的是结束后是不是clean up状态，我的理解做清理工作，有点类似于退出程序后清除所有文件
* -it 两个调试参数, 一般连在一起用，大致表示进入命令行交互输入输出的模式
* jekyll/minimal:$JEKYLL_VERSION 表示远程的镜像名称
* --volume=$PWD:/srv/jekyll 这句话含义比较多，首先--volume是挂载卷，比如把本地的主机目录挂载到docker下，用分号个开始的是两绝对个目录路径，前者是主机目录，后者是docker镜像实例中的目录。具体来说这个是把当前目录用pwd命令输出，并作为主机文件目录的挂载目标；而/srv/jekyll是一个预定义的目录，是jekyll/jekyll镜像内提前约定的一个文件位置；仅此而已；可以通过其Dockerfile配置源文件印证；
* jekyll build 最后是执行命令，编译得到静态站点

# 遇到的问题

好，我们理解了这里面的参数，那么到底怎么才能成功运行呢，笔者尝试了standard和minimal都没有成功；

![nokogiri-install-error.png]({{ site.baseurl }}/assets/images/nokogiri-install-error.png)

这个提示很郁闷，但是笔者明确记得在另一台mac上相同配置是没有问题的；尝试清除Gemfile.lock，重新获取依赖关系；遇到了另一个问题，ruby访问太慢了，比蜗牛还夸张，因此，只能更换国内镜像：

![ruby-mirro.png]({{ site.baseurl }}/assets/images/ruby-mirro.png)

换了镜像之后，速度上来了，但是任然有问题；只能把目光放到wiki上寻找答案；

![jekyll-docker-pages]({{ site.baseurl }}/assets/images/jekyll-docker-pages.png)

也就是除了三个常规的镜像，还有一套pages，所以这成了我们新的尝试点，把版本号改成pages后，成功执行了bundle install没有错误提示，只是在jekyll serve时报了一个依赖库找不到，但实际上这个依赖库明明已经安装了；

由于我们到现在都是以-it的形式执行，出错后直接退出，导致也没办法看更多信息；

回想一下，既然docker是容器，安装了linux环境，那么应该是可以登录进去做操作的才对。

![run-in-bash.png]({{ site.baseurl }}/assets/images/run-in-bash.png)

现在我们登录了docker容器实例，可以像正常使终端那样查看文件结构和安装的软件，先看一下刚才提示找不到的依赖包：

![coffie-script-source-duplicate.png]({{ site.baseurl }}/assets/images/coffie-script-source-duplicate.png)

可以看到确实是安装了的，但是有两份，会不会是这个原因导致的？试着卸载其中的低版本

![uninstall-duplicate-coffee-script.png]({{ site.baseurl }}/assets/images/uninstall-duplicate-coffee-script.png)

最后，在命令行下终于成功jekyll serve起来了。

# 小结

由于不熟悉Docker，在迁移过程中有很多细节都是很模糊的，好在docker的命令入门不难，还是要多加阅读；






