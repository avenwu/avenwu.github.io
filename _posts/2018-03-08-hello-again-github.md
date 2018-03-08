---
layout: post
title: "GitHub/SSH 故障"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2018-03-08-01.png
keywords: "GitHub，ssh"
tags: []
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2018-03-08-01.png)

## 背景

这两天GitHub开始抽风，稳定使用了多年的ssh配置，突然连接不上GitHub，每次同步代码有提示要输入密码，如果要手工输入密码的话，配置ssh还有什么意义？

事故时间比较巧，恰好这几天升级了系统，又适逢GitHub经历史上最强DDoS攻击（1.35T）。

## 故障排查

* 首显现确认ssh是否正常？

> 由于其他同样经由ssh配置的仓库任然可以正常访问，仅仅是GitHub上的所有仓库访问要求秘钥；问题可以缩小为GitHub的配置

本机为了方便，针对不同平台，配置了不同的rsa秘钥对，并利用config实现对不同Host的指向。经检查rsa秘钥均未丢失。
GitHub后台ssh配置也正常，没有变化。

查阅GitHub提供的SSH检测，进行本地排查
[Reviewing your SSH keys](https://help.github.com/articles/reviewing-your-ssh-keys/)


## 重新配置ssh

* ssh检查

```shell
eval "$(ssh-agent -s)"
Agent pid 59566
```

```shell
ssh-add -l -E md5
2048 MD5:a0:dd:42:3c:5a:9d:e4:2a:21:52:4e:78:07:6e:c8:4d /Users/USERNAME/.ssh/id_rsa (RSA)
```

如果ssh-add -l 提示没有已添加的rsa秘钥，那么可以手工添加

```shell
ssh-add -K ~/.ssh/id_rsa
```

其中id_rsa是你的的秘钥文件，名字不一定相同。

最后在config中修改配置

```
Host github.com
 AddKeysToAgent yes
 UseKeychain yes
 IdentityFile ~/.ssh/id_rsa
```
重启终端，回到git仓库，git pull，首次可能任然要输入密码，后续则恢复正常；
