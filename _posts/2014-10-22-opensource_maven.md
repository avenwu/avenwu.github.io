---
layout: post
title: "向Maven仓库提交开源项目"
tagline: ""
tags : [opensource,maven]
---
{% include JB/setup %}

maven库中存在大量开源的项目，那么如何提交一个Android开源库到meven？

通过maven官方文档，可以知道maven库有挺多机构支持向其中提交项目，对个人开发者来说 [sonatype](https://oss.sonatype.org/#welcome) 是最直接的选择。注册两个账号，sonatype和他的 [jira账号](https://issues.sonatype.org/secure/Dashboard.jspa)，jira主要适用于申请新项目的，需要先开一个jira问题，填写上项目的地址和你申请的唯一标识group id，可以用你的网址域名，没有的话用github上的你的地址，总之就是这个域名必须是你拥有的，如com.github.zhangsan, 一旦申请成功发布的项目可以以这个域名或她的子域名做标识，具体操作细节跟着文档指引即可。

比较有意思的是和jira上的人沟通，如果你填写的内容有任何问题，他们会恢复你，并且指出问题，提供可选建议，通过后就要配置本地项目，以使其支持向sonatype推送代码，简单起见可以引用chrisebanes的gradle文件， apply from: 'https://raw.github.com/chrisbanes/gradle-mvn-push/master/gradle-mvn-push.gradle'
这个文件是用groovy写配置信息，gradle将会据此生成maven的pom文件，具体细节不必深究，支持在本地的gradle.properies中填写从sonatype出得来的账号密码，其他的项目地址，开发者，gpg加密什么的都是根据自己的实际情况来写，所以还需要生成gpg签名文件，google之。

最后将项目推送至远程： `./gradlew uploadArchives`

最后给一个参考案例，[IndexImageView](https://github.com/avenwu/IndexImageView)这个是一个ImageView的Android空件，用于显示圆形图片，添加角标什么的。