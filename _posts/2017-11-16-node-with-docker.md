---
layout: post
title: "Node部署到Docker镜像"
description: "将node应用部署到docker镜像示例"
header_image: /assets/img/2017-11-16-02.jpg
keywords: "node, docker"
tags: [node]
---
{% include JB/setup %}
![img](/assets/img/2017-11-16-02.jpg)

## 背景

在用上了docker之后，本机就可以减少安装各种场景需要的软件，从而避免后续的卸载，更新，冲突之类的。
之前讲过把jekyll服务部署到docker中，这次来聊聊nodejs部署到docker。


## node程序

为了演示，我们需要先创建一个简单的node程序，下面我们写一段js他会让在被请求的时候返回`Hello world`

* server.js

```
'use strict';

const express = require('express');

// Constants
const PORT = 8080;
const HOST = '0.0.0.0';

// App
const app = express();
app.get('/', (req, res) => {
  res.send('Hello world\n');
});

app.listen(PORT, HOST);
console.log(`Running on http://${HOST}:${PORT}`);
```

同时我们把package.json修改如下：

```
{
  "name": "docker_web_app",
  "version": "1.0.0",
  "description": "Node.js on Docker",
  "author": "First Last <first.last@example.com>",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.13.3"
  }
}
```
里面的name，verion，author之类的可以自行修改。

现在你可以直接通过`npm start`启动程序了。

并且当你请求8080端口时，会得到一个`Hello world`的响应。

## docker配置

接下来，我们需要选用一个合适的docker镜像，这个可以google一下，node官方也提供了很多的镜像可以选用[Node Docker Images](https://hub.docker.com/_/node/)。

下面我们将参照[Dockerizing a Node.js web app](https://nodejs.org/en/docs/guides/nodejs-docker-webapp/) 一文，来从头开始把一个node程序部署到docker。

首先创建一个`Dockerfile`,

```
FROM node:alpine

# Create app directory
WORKDIR /usr/src/app

# Install app dependencies
COPY package.json .
# For npm@5 or later, copy package-lock.json as well
# COPY package.json package-lock.json ./

RUN npm install
# If you are building your code for production
# RUN npm install --only=production

# Bundle app source
COPY . .

EXPOSE 8080
CMD [ "npm", "start" ]
```

这里我们选用的是基于alpine来作为初始系统环境，只是因为这个Linux的发行版本提交很小，这样就不会占用我们过多磁盘空间。

Dockerfile相当于一个镜像的配置文件，其中每一行都有对应的注释。接下来我们需要build生成一个镜像。

```
docker build -t <your username>/node-web-app .
```
其中的`<your username>/node-web-app`写成你自己想要的名字就可以，比如`zhangsan/app`都是可以的。

## 运行镜像

现在就已经可以运行刚才创建的镜像了。

```
docker run -p 49160:8080 -d <your username>/node-web-app
```

我们通过-p参数，指定本机的49160端口，映射到docker中的8080端口。-d的含义是让这个docker以后台形式运行起来

一旦docker镜像跑起来后，我们可以继续通过终端查询当前的运行实例

```
docker ps
```

```
aven-mac-pro-2:AppDemo aven$ docker ps
CONTAINER ID        IMAGE                 COMMAND             CREATED             STATUS              PORTS                                 NAMES
172d309f6b4e        avenwu/node-web-app   "npm start"         36 minutes ago      Up 36 minutes       0.0.0.0:49160->8080/tcp               heuristic_golick
```

现在可以通过浏览器访问与喜爱49160端口,也可以通过终端请求

```
aven-mac-pro-2:AppDemo aven$ curl -i localhost:49160
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: text/html; charset=utf-8
Content-Length: 12
ETag: W/"c-M6tWOb/Y57lesdjQuHeB1P/qTV0"
Date: Thu, 16 Nov 2017 08:17:32 GMT
Connection: keep-alive

Hello world

```
## 小结

可以看到把一个node服务部署到docker中是比较方便的，但是这样的操作并不太合适开发，因为我们在创建Dockerfile的时候其实已经把源码内容都固定了，所以在部署以后再改动源代码是不会生效的，当然要解决整个也是可以的，我们只需要通过volume进行磁盘映射即可。

