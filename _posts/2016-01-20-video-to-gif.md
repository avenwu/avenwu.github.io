---
layout: post
title: "自己动手，视频转GIF"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-04.jpg
keywords: "gif 转换"
description: "video to gif"
category: 
tags: [视频]
---
{% include JB/setup %}

![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-04.jpg)

## 前言

经常需要生成GIF图片，Google可以找到不少软件，但没有比较好用的免费软件，gif brewery好用但是收费；

有没有办法自己手工实现简单的转换？当然有

[http://www.schneems.com/post/41104255619/use-gifs-in-your-pull-request-for-good-not-evil/](http://www.schneems.com/post/41104255619/use-gifs-in-your-pull-request-for-good-not-evil/)

文章作者利用了开源的ffmpeg，将视频导成gif，再通过gifsicle调整gif；

## 开始DIY

1. 首先需要安装ffmpeg和gifsicle；安装方法很多，这里笔者用的是brew安装

		brew install ffmpeg
		brew install gifsicle

2. 安装完工具就可以开始转换视频了；

		ffmpeg -i in.mov -pix_fmt rgb24 -f gif - | gifsicle -O3 -d3 > out.gif  

3. 如果文件比较多可以写个脚本，自动将视频批量转换, 下面的脚本会将当前目录的所有MP4文件转换为GIF

{% highlight bash %}
#!/usr/bin/env bash
for file in ./*mp4
do
	echo "Convert $file"
	ffmpeg -i $file -s 360x640 -pix_fmt rgb24 -r 10 -f gif - | gifsicle --optimize=3 --delay=20 > "$file.gif"
done
{% endhighlight %}

关于ffmepg的更多用法可以自行google；

## Reference
* [https://www.reddit.com/r/ruby/comments/16zvjt/use_gifs_in_your_pull_request_for_good_not_evil/c81btvm](https://www.reddit.com/r/ruby/comments/16zvjt/use_gifs_in_your_pull_request_for_good_not_evil/c81btvm)

* [http://www.schneems.com/post/41104255619/use-gifs-in-your-pull-request-for-good-not-evil/](http://www.schneems.com/post/41104255619/use-gifs-in-your-pull-request-for-good-not-evil/)
