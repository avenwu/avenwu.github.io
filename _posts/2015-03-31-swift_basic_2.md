---
layout: post
title: "[swift]操作符重载"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-22.jpg
description: "swift语法基础"
category: "swift"
tags: [swift]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-2.jpg)

做c++开发的都知道，再c++里面是允许重载运算符的，在swift中也可以重载运算符，下面来重载一下++，+
上上一篇中曾今定义了一个MyClass类，现在我们通过重载++和+使得MyClass的实例可以向数值型变量一样进行相加和自增操作。

重载在形式上非常简单就和定义一个方法一样

{% highlight swift %}

class MyClass {
    var money = 100 as Int
    init() {
        
    }
    init(money : Int) {
        self.money = money
    }
    func addMonet(count : Int){
        money += count;
    }
}

/**
* 重载++,运算符在变量后
**/
postfix func ++ (inout myClass : MyClass) -> MyClass {
    var newClass = MyClass(money : myClass.money)
    myClass.money++
    return newClass
}
/**
* 重载++,运算符在变量前
**/
prefix func ++ (inout myClass : MyClass) -> MyClass {
    myClass.money++
    var newClass = MyClass(money : myClass.money)
    return newClass
}

/**
* 重载+
**/
func + (left : MyClass, right : MyClass) -> MyClass {
    var newClass = MyClass(money : left.money + right.money)
    return newClass
}

{% endhighlight %}


现在,可以直接对MyClass的实例进行自增和相加操作了。
{% highlight swift %}

let newClass = myClass++
myClass
let newClass2 = ++myClass

var a = 3
a++
++a
let class3 = newClass+newClass2
{% endhighlight %}

---