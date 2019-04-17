---
layout: post
title: "比较之外，计数排序"
description: "计数排序counting sort"
header_image: /assets/img/2016-03-28-02.jpg
keywords: "计数排序 counting sort"
category: 
tags: [算法]
---
{% include JB/setup %}
![img](/assets/img/2016-03-28-02.jpg)

## 前言
陆续介绍了好几种排序，这些都是基于比较数值的比较排序，今天看一看另一种基于计数的排序算法，也算别有洞天。

## 不比较
假设待排序的是正整数范围内的数的集合（稍加换算也可以使用与负数范围），计数排序的显著特点就是绕过了排序时两两比较大小的过程，其主体思路是，引入一个中间数组C，将数值映射到改中间数组C的下标，元素value在C内的位置就是C[value]，而C[value]这个位置存储的值在排序中不断变化，一方面可以保存v元素alue重复的个数；

	* C[value]=0表示待排序数组中不存在value元素；
	* C[value]=n表示待排序数组存在n个元素value；
需要注意的是C在不同阶段存储不同的值，表示不同含义，第二个阶段，基于C[value]表示value的个数，我们可以进一步推理出value应该在排好序后的数组中存放的位置

	C[value] = C[value] + C[value-1]
在运用这个公式得到value存放位置时必须确保是自增的循环处理，举个例子：

	C[1] = C[1] + C[0]
	元素1应该存储的位置必去确保前一个元素0顺序排好，0暂用了C[0]个位置，C[1]也可能重复，所以C[1]的位置是C[1]个加上C[0]位置；

由此两步，我们成功地借助于C数组存储了数据的排序位置；最后只需要倒序将C内的的位置一一输出到用于存放排好序的数组B中；


	记住这里一定是倒序遍历的，而不能是正序遍历，为什么呢？

因为每个元素是可能重复的，必须确保重复的元素得到正确的存储位置，所以倒序遍历时，还需要将C内对应的数量自减；

{% highlight swift %}
#!/usr/bin/env swift
/**
* 计数排序
*/

//A数据源，B排序结果，C辅助计数数组

func countingSort(inout A:[Int], inout B:[Int], k:Int) {
    var C = [Int](count:k+1, repeatedValue:0)
    for var i = 0; i < A.count; i++ {
        C[A[i]] = C[A[i]] + 1
    }
    println("C element store count, C=\(C)")

    for var j = 1; j <= k; j++ {
        C[j] = C[j] + C[j-1]
    }
    println("C element store index, C=\(C)")
    for var i = A.count - 1; i > 0; i-- {
       
        println("i=\(i)")
        var value = A[i]
        var index = C[value] - 1
        B[index] = value
        C[value]--
        println("C=\(C)")
        println("B=\(B)")
    }
}

var A = [2, 5, 3, 0, 2, 3, 0, 3]
var B = [Int](count:A.count, repeatedValue:0)
println("Array:\(A)")
var k = A[0]
for i in A {
    if i > k {
        k = i
    }
}
println("k=\(k)")
countingSort(&A, &B, k)
println("Sorted:\(B)")

{% endhighlight %}

## 小结
初次看到计数排序感觉非常有意思，巧妙的利用下标映射数据，实现了数据的计数表示；理解计数排序的关键是理解辅助数组存放的东西是变化的，最终是为了得到下标对应元素需要放置的位置；


