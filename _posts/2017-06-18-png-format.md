---
layout: post
title: "从魔术字开始分析png的构造"
description: ""
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2017-06-18-01.jpg
keywords: ""
tags: [PNG]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2017-06-18-01.jpg)

## 前言

图片在在互联网开发中的重要性已经渗入各个角落，而PNG在移动端的普及更不在话下，无论是Android，iOS还是Web端，虽然也有压缩率更好的webp出现，不过webp不在本文讨论范围；

下面一起从PNG的图片格式开始，一步步了解一张PNG都包含哪些内容，以及如何读取相关信息；

## PNG格式规范

任何文件都有它的格式规范，根据相关资料在PNG之前他的前身其实一种受专利保护的一种LZW压缩算法，PNG是在此基础之上提出来的一种图片格式，全称是Portable Network Graphic Format
PNG是是无损压缩的一种图片格式，并且支持透明通道，根据每个像素用多少未来表示颜色，又可以做细分；一般来说图片相对占用空间更大；

## 魔术字和他的伙伴们

图片在存储的时候都是一些二进制的基础信息，如果我们用一些文本编辑器或者Hex编辑器打开一张图片，经常会看到类似下面这样的一堆数字；

![png hex preview](http://7u2jir.com1.z0.glb.clouddn.com/img/png-hex-preview.png)

注意起始的一串数字，PNG总是以89 50 4e 47 0d 0a 1a 0a 这一串开头。

这是因为根据PNG文件格式约定，其开头的8个字节是PNG的签名字节。并且每一个字节都是有含义的。为了更容易理解，笔者绘制了几张图：

![png magic header](http://7u2jir.com1.z0.glb.clouddn.com/img/png-8bytes-signature.png)

这8个字节每一组的含义其实也都很实际，第一个是魔术字作用大家可以参考PNG规范[^1]，接着的三个字节就是PNG的字节表示，最后几个分别表示了不同系统下的换行符；

[^1]: [PNG格式规范](https://www.w3.org/TR/PNG/#11IHDR)

当然如果你不习惯看16进制的数据，也可以转换为十进制和ASCII来看，都是差不多的。后续我们在演示的时候实际上都是通过十六进制来表示，一方面不存在转义符，也不为出现位数问题，同时十六进制比较整齐；

![png magic header decimal](http://7u2jir.com1.z0.glb.clouddn.com/img/png-8bytes-signature-decimal.png)

![png magic header ascii](http://7u2jir.com1.z0.glb.clouddn.com/img/png-8bytes-signature-ascii-c.png)

## 图片头信息解析
签名头之后就是一些称之为chunk的块数据；这些chunk也按照一定的格式进行组织；

![png chunk format](http://7u2jir.com1.z0.glb.clouddn.com/img/png-chunk-format.png)

PNG的Chunk块具备较强的扩展性，在开始真正的像素数据块之前，我们熟知的图片宽度，长度，颜色模式等也都是放在chunk中的，根据约定，这些数据放在了第一个chunk中，被称为IHDR，类似还有PLTE,IDAT,IEND等，这些被称之为Critical Chunk，大意是格式要求严格的块。所有的png解码工具都必须能解析这些数据；

由于IHDR中有我们比较关心的数据，现在先看下IHDR的样子，首先他就是一个Chunk，所以肯定满足Chunk的四大块划分，同时IHDR是所包含的内容是确定，因此它的更多数据也是确定的，比如Length的具体长度就是13，类型是IHDR，Chunk Data中就是含义固定的13个字节，最后是CRC校验码。

所以我们直接看Chunk Data的13个字节。

![png chunk format](http://7u2jir.com1.z0.glb.clouddn.com/img/ihdr-format.png)

这13个字节中宽高，颜色类型，深度还能理解，另外一些字段的话不是很好理解，具体参考官方说明。

与Critical Chunk向呼应的是Ancillary Chunk，大意是附加块；其数量相比Critical来说就多很多了。

## 提取头信息

既然我们一直知道了一张png图片的头部构造和高宽的存储位置，我们就可以通过代码来校验一个文件是否是png图片；

首先根据8字节的签名，我们可以校验一个文件是否是png（更严格的最好再通过获取高宽）

```
int[] PNG_HEADER_ASCII = {0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a};

// ...

inputStream = new FileInputStream(file);
byte[] header = new byte[PNG_HEADER_ASCII.length];
int read = inputStream.read(header);
if (read != PNG_HEADER_ASCII.length) {
  System.out.println("Failed to get image header: read " + read + "bytes");
  return false;
}
for (int i = 0; i < PNG_HEADER_ASCII.length; i++) {
  //必须的，去高位，只保留两位十六进制结果
  int c = 0xFF & header[i];
  if (c != PNG_HEADER_ASCII[i]) {
    System.out.println("Miss match byte[" + i + "]=" + header[i]);
    return false;
  }
}

```

解析IHDR中的数据并输出：

```
// check IHDR
// length : 4bytes
byte[] lengthBytesArray = new byte[4];
read = inputStream.read(lengthBytesArray);
int l = bytes2int(lengthBytesArray);
System.out.println("lengthBytesArray=" + l);

// chunk type : 4bytes
// 73 72 68 82
byte[] chunkType = new byte[4];
read = inputStream.read(chunkType);

// length data
byte[] widthBytesArray = new byte[4];
// widthBytesArray
read = inputStream.read(widthBytesArray);
int width = bytes2int(widthBytesArray);
System.out.println("width:" + width);
//height
read = inputStream.read(widthBytesArray);
int height = bytes2int(widthBytesArray);
System.out.println("height:" + height);
```

## 未完待续

PNG的内容比较多，其他有意思的细节后续分享。