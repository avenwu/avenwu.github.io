---
layout: post
title: "unzip解压特定文件"
description: ""
header_image: /assets/img/2017-07-06-02.jpg
keywords: "unzip解压"
tags: [shell]
---
{% include JB/setup %}
![img](/assets/img/2017-07-06-02.jpg)

# 前言

在处理zip压缩包时，我们经常和unzip打交道，比如解压，压缩什么的，那么如何解压zip文件中的某一个文件呢？

带着这个疑问我们完整学习一下unzip的更强大的功能

# help vs man

unzip 的help可以告诉我们很多其支持的参数，但是这些说明并不是很容易理解，

```
aven-mac-pro-2:Desktop aven$ unzip --help
UnZip 5.52 of 28 February 2005, by Info-ZIP.  Maintained by C. Spieler.  Send
bug reports using http://www.info-zip.org/zip-bug.html; see README for details.

Usage: unzip [-Z] [-opts[modifiers]] file[.zip] [list] [-x xlist] [-d exdir]
  Default action is to extract files in list, except those in xlist, to exdir;
  file[.zip] may be a wildcard.  -Z => ZipInfo mode ("unzip -Z" for usage).

  -p  extract files to pipe, no messages     -l  list files (short format)
  -f  freshen existing files, create none    -t  test compressed archive data
  -u  update files, create if necessary      -z  display archive comment
  -x  exclude files that follow (in xlist)   -d  extract files into exdir

modifiers:                                   -q  quiet mode (-qq => quieter)
  -n  never overwrite existing files         -a  auto-convert any text files
  -o  overwrite files WITHOUT prompting      -aa treat ALL files as text
  -j  junk paths (do not make directories)   -v  be verbose/print version info
  -C  match filenames case-insensitively     -L  make (some) names lowercase
  -X  restore UID/GID info                   -V  retain VMS version numbers
  -K  keep setuid/setgid/tacky permissions   -M  pipe through "more" pager
Examples (see unzip.txt for more info):
  unzip data1 -x joe   => extract all files except joe from zipfile data1.zip
  unzip -p foo | more  => send contents of foo.zip via pipe into program more
  unzip -fo foo ReadMe => quietly replace existing ReadMe if archive file newer

```

常见的-d 指定目录，-p解压到管道，-l列出包内容,那其他一些呢，看到这么简略的参数介绍，简直无助；

比如-j 这里的意思解压文件忽略路径，那意味着整个压缩包内部的所有文件在解压的时候都不在创建对应具体目录，而是直接解压？

为了更好的弄明白这些参数，我们还需要借助man来查看更详细的说明文档

```
man unzip
```

整个信息就非常全了，所以再看了help后还是没解决疑惑的话，可以再用man查阅帮助文档；

# 解压特定文件

回到我们的主题，那么到底如何解压某一个文件出来？

## Google大法好

要解决整个问题可以google一下，一般可以找到答案，比如[extract-only-a-specific-file-from-a-zipped-archive-to-a-given-directory](https://unix.stackexchange.com/questions/14120/extract-only-a-specific-file-from-a-zipped-archive-to-a-given-directory), 

## unzip -j

文中提到了两种方法来解压一个文件，我们挑选了其中更好的一种方式

```
unzip -j "myarchive.zip" "in/archive/file.txt" -d "/path/to/unzip/to"
Enter full path for zipped file, not just the filename. Be sure to keep the structure as seen from within the zip file.

This will extract the single file file.txt in myarchive.zip to /path/to/unzip/to/file.txt.
```

简单来说就是利用-j来解压文件，然后指定要解压的文件在zip包内的具体路径，并指定解压的目标位置

比如我想看一下apk里面的签名清单文件, 注意不是AndroidManifest.xml，而是MANIFEST.MF

先用-l查一下他的完整路径：

```
aven-mac-pro-2:Desktop aven$ unzip -l 7.12.apk |grep META
      652  06-30-17 14:37   META-INF/rxjava.properties
   640127  06-30-17 14:37   META-INF/MANIFEST.MF
   640156  06-30-17 14:37   META-INF/CERT.SF
      891  06-30-17 14:37   META-INF/CERT.RSA
```

接着解压META-INF/MANIFEST.MF到本地

```
aven-mac-pro-2:test aven$ unzip -j 7.12.apk META-INF/MANIFEST.MF ./
Archive:  7.12.apk
  inflating: MANIFEST.MF             
caution: filename not matched:  ./
```
成功了，但是有个警告，这个是因为./我们没有用-d所以尝试指定目录是没有被识别，unzip还以为./是第二个文件，不过没关系，当我们的指定路径失败了，他会直接解压到道歉目录，也就正好是./目录


那么我们要解压多个特定文件可以么？

```
aven-mac-pro-2:test aven$ unzip -j 7.12.apk META-INF/MANIFEST.MF META-INF/*
Archive:  7.12.apk
  inflating: rxjava.properties       
  inflating: MANIFEST.MF             
  inflating: CERT.SF                 
  inflating: CERT.RSA     
```

可以看到，完全是没问题的，META-INF下的文件我们用通配符*全部解压到了当前文件夹下面：）666

```
aven-mac-pro-2:test aven$ ls -al
total 70592
drwxr-xr-x   7 aven  staff       238 Jul  5 13:36 .
drwx------@ 27 aven  staff       918 Jul  5 13:26 ..
-rw-r--r--@  1 aven  staff  34847354 Jul  5 13:27 7.12.apk
-rw-r--r--@  1 aven  staff       891 Jun 30 14:37 CERT.RSA
-rw-r--r--@  1 aven  staff    640156 Jun 30 14:37 CERT.SF
-rw-r--r--@  1 aven  staff    640127 Jun 30 14:37 MANIFEST.MF
-rw-r--r--@  1 aven  staff       652 Jun 30 14:37 rxjava.properties
```

## unzip -p

除了-j的话，上文还提到了-p，也就是输出到管道的操作

```
You can extract just the text to standard output with the -p option:

unzip -p myarchive.zip path/to/zipped/file.txt >file.txt
This won't extract the metadata (date, permissions, …), only the file contents. That's the price to pay for the convenience of not having to move the file afterwards.

Alternatively, mount the archive as a directory and just copy the file. With AVFS:

mountavfs
cp -p ~/.avfs"$PWD/myarchive.zip#"/path/to/zipped/file.txt .
Or with fuse-zip:

mkdir myarchive.d
fuse-zip myarchive.zip myarchive.d
cp -p myarchive.d/path/to/zipped/file.txt .
fusermount -u myarchive.d; rmdir myarchive.d

```

正如作者表述的那样，-p实际上是吧内容输出到管道，然后我们再用>重定向到文件，所以解压出来的实质文件内容，其他文件属性目都会缺失；

```
aven-mac-pro-2:test aven$ unzip -p 7.12.apk META-INF/MANIFEST.MF >MANIFEST.MF
aven-mac-pro-2:test aven$ ls -al
total 69320
drwxr-xr-x   4 aven  staff       136 Jul  5 13:47 .
drwx------@ 27 aven  staff       918 Jul  5 13:26 ..
-rw-r--r--@  1 aven  staff  34847354 Jul  5 13:27 7.12.apk
-rw-r--r--   1 aven  staff    640127 Jul  5 13:47 MANIFEST.MF
```

这个方法相比-j的话有几个劣势：

* 有一点麻烦，需要指定输出的名字；
* 并且仔细观察的话，可以看到文件的创建时间确实变了；
* 不能同时操作多个特定文件；

如果单纯的导出一个文件的话两种方法都是可行的，如果是直接在屏幕输出，还是-p更方便些；

# 放开Google，让我来

搜索来的方法固然很快，但是如果不搜索的话，针对这些近在眼前的命令工具，有没有办法自食其力，或者说并不可能每次你想要的操作都正好那么快被搜索到的时候怎么办呢？

下面我们一起用man来实际分析一下一个命令的帮助文档如何帮助我们；

## man

man是最好用的工具，对不熟悉的系统命令，都可以尝试用man来获取一下帮助信息

```
aven-mac-pro-2:Desktop aven$ man unzip

UNZIP(1L)                                                                                                 UNZIP(1L)

NAME
       unzip - list, test and extract compressed files in a ZIP archive

SYNOPSIS
       unzip [-Z] [-cflptTuvz[abjnoqsCKLMVWX$/:]] file[.zip] [file(s) ...]  [-x xfile(s) ...] [-d exdir]

```

这是前两行，我们得知两个信息：

1. unzip用于操作ZIP归档文件，可以展示(list)，测试（test），解压（extract）ZIP内的压缩文件
2. unzip所支持的参数写法，[]一般都是表示可选，这里也是一个意思；

整个帮助信息，比较多，定位到-j和相关的示例

```
       -j     junk paths.  The archive's directory structure is not recreated;  all  files  are  deposited  in  the
              extraction directory (by default, the current one).

EXAMPLES
       To use unzip to extract all members of the archive letters.zip into the current directory and subdirectories
       below it, creating any subdirectories as necessary:

           unzip letters

       To extract all members of letters.zip into the current directory only:

           unzip -j letters
```

结合前面的使用参数位置说明，我们只需要指定具体[file(s) ...]便可以解压特定文件；


# 小结

命令很多，及时记不住全部的，借助线索查找，并记忆常用的参数可以提高工作效率；







