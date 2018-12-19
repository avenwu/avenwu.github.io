---
layout: post
title: "低成本“全栈”回忆录"
description: ""
header_image: "/assets/images/react/web-preview.png"
keywords: "react"
tags: [react]
---
{% include JB/setup %}
![img](/assets/images/react/web-preview.png)


> “建站”是一个很古老的话题了，互联网有各种快速建站的小广告。

相信每位开发者都有过各种倒腾的经历，随着大量平台工具的兴起，建个开发者博客什么的，早已旧时王谢堂前燕，飞入寻常百姓家。而移动端开发者学习/掌握大前端技术体系是趋势所在，了解前后端/运维等知识也多有裨益。

在Android部门内，开发维护了一些平台工具，目前已经基本形成一套完整的工具链，既有[在线工具集](http://10.9.188.58:8888/)，也有[AVM平台](http://andr.avm.58v5.cn)，[58APP协议平台](http://protocol.58v5.cn/)。在参与开发这些项目的过程中，有幸为这些不同框架体系的项目添砖加瓦。

> 那么如何`以较低的开发/学习成本，开发一个完整站点并上线`是这篇文章将要介绍的。

本文基于笔者的一些倒腾案例总结而来，包教不会，不正确的地方欢迎交流/指正。

## 0x1 项目构思

作为一枚宅男，看影视剧是一个难以割舍的爱好，英剧/美剧/韩剧都有很多吸引人的地方。

作为一枚技术宅，是不是可以自己搞一个小电影的网站/客户端？

学习研究一个东西肯定是要做一些平时做的少的东西，才有满足感；我们跳过客户端开发，定个小目标：

> 做一个适配手机浏览器的小网站，能展示电影列表信息，并且点击下载资源。

[Record_2018-12-19-14-40-45.mp4](/assets/files/Record_2018-12-19-14-40-45.mp4)

完成这个一句话目标，我们需要考虑的事情可以用一张图来概括：

![小站电影-脑图](/assets/images/react/小站电影-脑图.png)

下面我们顺着思维脑图，逐一介绍下整个开发历程。

## 0x2 前端开发

首先整个网站的前端是一个独立项目。我们走的是React技术路线，通过[create-react-app](https://facebook.github.io/create-react-app/)这个脚手架工具来创建前端工程。不要问我为什么，作为新手不用这个的话，大量新的框架概念术语，会让你的脑容量瞬间不够用。

这个脚手架的说明介绍也很干净利落：

* Less to Learn
* Only One Dependency
* No Lock-In

![create-react-app](/assets/images/react/create-react-app.png)

### 0x2.1 模板工程

成熟的框架，都会有一套模板工程，React当然不例外。它的发车姿势比较简单，真的是一个命令搞定：`npx create-react-app my-app`

![create-project](/assets/images/react/create-project.svg)

有了模板工程以后，就可以开始写你的业务界面了。

为了让我们的页面看起来更美观&专业，强力建议选择一个UI组件库，`Bootstrap，Material UI`都可以，毕竟这是一个看脸的世界。这里我们选择的是[Semantic UI React](https://react.semantic-ui.com/), 长得好看还好用

### 0x2.2 页面开发

用react写界面，大部分工作都在构造Component，一个Component写法如下:

```jsx
class SegmentExamplePlaceholderGrid extends React.Component {
  render(){
    return (<div></div>)
  }
}
export default  SegmentExamplePlaceholderGrid
```

如果你的Component非常简单，也不需要实现任何生命周期，只有一个基本的`render`，也可以简写。举个例子，如果要实现下面这个效果，利用Sementic UI组件，可以这么写：

![react-sample-1](/assets/images/react/react-sample-1.png)

```jsx
import React from 'react'
import { Button, Divider, Grid, Header, Icon, Search, Segment } from 'semantic-ui-react'

const SegmentExamplePlaceholderGrid = () => (
  <Segment placeholder>
    <Grid columns={2} stackable textAlign='center'>
      <Divider vertical>Or</Divider>

      <Grid.Row verticalAlign='middle'>
        <Grid.Column>
          <Header icon>
            <Icon name='search' />
            Find Country
          </Header>

          <Search placeholder='Search countries...' />
        </Grid.Column>

        <Grid.Column>
          <Header icon>
            <Icon name='world' />
            Add New Country
          </Header>
          <Button primary>Create</Button>
        </Grid.Column>
      </Grid.Row>
    </Grid>
  </Segment>
)

export default SegmentExamplePlaceholderGrid
```

"小站电影"项目，这里这里写的是一个List,展示一个电影列表：

![web-preview](/assets/images/react/web-preview.png)

实现长列表，我们使用的是`react-virtualized`，类似于RecycleView可以做单元复用，避免li直接展示所有数据；

### Route

主要界面有两个，一个是主页，包含侧滑菜单和列表页，导航栏，还有一个就是电影的详情页，两个页面之间的跳转，我们使用Route来实现：

```jsx
import React from "react";
import { BrowserRouter as Router, Route, Link } from "react-router-dom";

function BasicExample() {
  return (
    <Router>
      <div>
        <ul>
          <li>
            <Link to="/">Home</Link>
          </li>
          <li>
            <Link to="/about">About</Link>
          </li>
          <li>
            <Link to="/topics">Topics</Link>
          </li>
        </ul>

        <hr />

        <Route exact path="/" component={Home} />
        <Route path="/about" component={About} />
        <Route path="/topics" component={Topics} />
      </div>
    </Router>
  );
}
```

## 0x3 后端开发

前端页面有了，我们需要为页面提供数据。那么我们的服务器用什么来做呢？

为了上手快，降低门槛，可以使用Node作为后端服务器的运行环境，这样可以直接使用Node的http模块，也可以使用`express`之类的框架。

```js
const express = require('express')
const app = express()
const port = 3000

app.get('/', (req, res) => res.send('Hello Express!'))

app.listen(port, () => console.log(`Example app listening on port ${port}!`))
```

类似的，我们可以开发真实的API，并且测试接口的数据返回情况。这里我们通过curl来简单测试下：

```shell
aven-mac-pro-2:~ aven$curl localhost:3001/api/movie/list/9/0 |jq .
```

我门的API是以json格式返回的，为了格式化看得清晰点，可以使用jq做输出。

![api-sample-1](/assets/images/react/api-sample-1.png)

## 0x4 数据采集

按部就班，我们的接口要返回数据，但是数据又从哪里来？

接下来就是如何去找数据。互联网上资源非常多，随便找找就有一堆，这里可以考虑下pv比较大的电影天堂。

### 0x4.1 xposed

电影天堂上有很多免费的良心资源，还有客户端app，忽略页面比较渣，还是能用的。抓包一下可以看到他的请求情况：

![api-sample-3](/assets/images/react/api-sample-3.png)

![api-sample-2](/assets/images/react/api-sample-2.png)

如果我们直接请求模拟这个接口请求，会失败的，通过观察可以发现`header`里面有两个字段比较可疑：

```
x-header-request-imei: 89014103211118510720
x-header-request-key: e62641ff809149a261ac7db336deb399
```

进一步反编译app，可以知道这个`x-header-request-key`是个加密串，

```java
 public final native String headerRequestKey(Context context, String str, String str2);
```

下面我们就模拟一下，加入加密的自定义头：

![api-sample-4](/assets/images/react/api-sample-4.png)

同时需要开发两个xposed模块，对应用进行签名拦截，hook做两件事：

1. 获取正版应用的签名字节数据
2. 替换测试应用的签名

![hook-1](/assets/images/react/hook-1.png)

![hook-2](/assets/images/react/hook-2.png)

接下来就可以开始起飞了，要注意，接口调用不要太频繁，不然ip会被封的，也不要问我是怎么知道：）

关于如何安装xposed框架，开发module不在我们的讨论范围之内，感兴趣可以去深入学习下：

> [de.robv.android.xposed.installer](https://repo.xposed.info/module/de.robv.android.xposed.installer)

### 0x4.2 网页爬虫

除了客户端破解，也可以考虑去扒一下它的网页，这样不需要反编译什么的，只需要好写好脚本去分析目标html。这个网上已经有人做过这件事情，就不再介绍，除了爬虫其实还有一个办法，去找一些资源采集器，用别人开发好的采集工具批量拉取数据。

## 0x5 服务器搭建

让我们回顾一下，现在数据也有了，需要考虑下一步，怎么把我们的服务和网站提供给外网用户访问，总不可能一直的本机跑吧。

如果手上有服务器的话可以直接用，不需要单独再买一个。没有的话，可以采购一台，国内的阿里云也不错，配置不用太好，选个便宜的，单核1G内存就够玩了。

> 1 CPU, 25G Storage, 1G RAM

为了好好利用和管理手上的机器，可以使用docker来部署应用，这样可以保证服务迁移和主机环境相互不影响。避免直接部署在主机上后，需要安装一堆的运行环境和依赖配置。

我们内部的平台工具目前使用的是[Portainer](https://portainer.io/)来管理docker实例，使用比较简单，功能也足够强大，我的主机也采用它来管理服务。下面是portainer的官方使用案例：

```shell
$ docker volume create portainer_data
$ docker run -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer
```

部署完服务，就可以登陆后台管理。新增，删除，重启docker镜像都可以直接操作。

![portainer-login](/assets/images/react/portainer-login.png)

这里我们新增了一个Node镜像，由于没有什么特殊依赖，直接使用了官方的镜像。启动镜像后，远程登陆然后可以开始你的骚操作，比如更新代码，启动脚本什么的。当然也可以把这些所有流程写入Dockerfile，构建我们自己的镜像，这样每次实例化对象，启动后不需要单独操作。

![portainer-manager](/assets/images/react/portainer-manager.png)



## 0x6 部署上线

现在万事具备只欠东风！

分配一个域名和我们的主机ip绑定。这样广大人民群众就可以统一域名访问我们的主机。

这里需要确定docker的端口映射，让主机80端口映射到docker实例的8080端口，在docker内部，我们的网站只需要监听8080端口就可以对外提供服务了。

为甚是80而不是443或者其他接口呢？这主要是因为我没有提供https接口，80端口是http协议默认端口，浏览器都认，如此所有请求`http://io.hacktons.cn`的等同于`http://io.hacktons.cn:80`

平时开发都是在本地，执行npm start就能跑起项目，如今要上线了，自然后准备一下脚本和配置，确保本地开发使用开发端口，上线使用上线端口，并且前端代码是上线的模式：

react-script为我们做了很好的封装，只需执行不同命令，带上参数配置：

```json
"scripts": {
  "start": "react-scripts start",
  "build": "GENERATE_SOURCEMAP=false react-scripts build",
  "test": "react-scripts test --env=jsdom",
  "eject": "react-scripts eject"
}
```

发布的时候走build命令，不生成source map： `GENERATE_SOURCEMAP=false react-scripts build`

我们看到所有源码经过webpack打包之后全部混淆，集中到了几个chunk/bundle文件内置：

![deploy-1](/assets/images/react/deploy-1.png)

后端代码由于不会暴露给前端，暂时没有什么特殊操作，只需要考虑端口号：

```json
"scripts": {
  "server": "node server.js",
  "prod":"PORT=8080 NODE_ENV=production npm run server"
}
```

我们可以吧所有要做的配置固化后，写成脚本，自动/手动执行：

![deploy-2](/assets/images/react/deploy-2.png)

## 0x7 总结

现在点击访问[io.hacktons.cn](http://io.hacktons.cn)就可以看到我们的小网页了。

讲一个段子：JavaScript是最好的语言，我不是针对PHP

⚠️本项目仅做技术研究，涉及的电影资源严禁非法下载传播。

------

**参考**

* https://www.fullstackreact.com/articles/using-create-react-app-with-a-server/
* https://facebook.github.io/create-react-app/docs/proxying-api-requests-in-development


* https://github.com/facebook/create-react-app/issues/1004


* https://github.com/facebook/create-react-app/blob/master/packages/react-scripts/template/README.md
