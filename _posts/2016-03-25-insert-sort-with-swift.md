---
layout: post
title: "算法之美，用swift写个插入排序"
description: "插入排序insert sort with swift"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-25-01.jpg
keywords: "insert sort"
category: 
tags: [算法 插入排序]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-25-01.jpg)

## 前言
	
	主说，要有光，于是就有了光

我不是虔诚的教徒，但是我也喜欢圣经里教人向善的普世良言。


## 举个栗子，再也忘不了插排

本文来聊聊更易理解的插入排序。之所以叫插入排序，不是为别的，正是因为该算法的核心就是将无序的元素插入排好序的部分。

这里有张我觉得很形象的的例子，即打牌是摸牌的过程（取自算法导论）

![img]({{ site.baseurl }}/assets/images/insert_poker.png)


	插入排序的核心思想即在于划分已排序和未排序，将每个待排序的元素逐个与已排序的元素比较，找出恰当的插入位置，插入元素，循环操作至结束

## 算法实现

这里是一张插入排序的使用流程，在写代码前先感受一下。

![img]({{ site.baseurl }}/assets/images/insert-sort.jpg)

我们以一维数组作为待排序的数据源，整个数组的以第一个待排序的元素为分水岭，前半部分为已排好序的，后半部分是等待排序的；
开始排序时，从第二个元素开始循环开始，由于需要记录当前待排序的元素，我们引入一个变量记录分水岭的下标，也就是下面源码内的变量i；
比较的过程比较直接，从分水岭往前，逐一比较值的大小，没找到需要插入的位置时向后移动元素，知道找到位置插入元素；

{% highlight swift %}
#!/usr/bin/env swift
/**
* 插入排序
*/

var intergerArray = [3,1,6,45,58,3,89,40]

println("intergerArray=\(intergerArray)")
for var j = 1; j < intergerArray.count; j++ {
    let key = intergerArray[j]
    var i = j-1
    while i >= 0 && intergerArray[i] > key {
        intergerArray[i+1] = intergerArray[i]
        i--
    }
    intergerArray[i+1] = key
}

println("Sorted:\(intergerArray)")
{% endhighlight %}

{% highlight shell %}
avens-MacBook-Pro:algrithm-swift aven$ ./insert-sort.swift 
Insert sort...
intergerArray=[3, 1, 6, 45, 58, 3, 89, 40]
Sorted:[1, 3, 3, 6, 40, 45, 58, 89]
{% endhighlight %}

## 小结
人生在世及时行乐，下面就没有啦，本文结束。

