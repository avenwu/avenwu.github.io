---
layout: post
title: "swift不一样的函数声明与使用"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-18.jpg
description: ""
category: 
tags: [swift]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-18.jpg)

swift中函数的声明有很多形式，主要集中在对形参的表示上面。

java中方法只有一个名字，没有扩展参数名，但是swift（oc）支持扩展方法名，也可以为参数设置默认值，同时支持方法类型的参数，还有一点就是支持闭包closure。

下面以hello为例，不同写法的效果是一样的，都返回一个Hello Frank的字符串。

{% highlight swift %}
//方法定义
//1.
func hello(n : String) -> String {
    return "Hello \(n)"
}

hello("Frank")
//2.扩展方法名name
func hello2(name n:String) -> String {
    return "Hello \(n)"
}
hello2(name:"Frank")

//3.形参和扩展方法名一样，name
func hello3(#name:String) -> String {
    return "Hello \(name)"
}
hello3(name: "Franck")

//4.为形参设置默认值
func hello4(name: String = "Franck") -> String {
    return "Hello \(name)"
}
hello4()

//5.声明变量形参
func hello5(var name: String = "Frank") -> String {
    return "Hello \(name)"
}
hello5()

//6.方法参数
func sayHello(name:String) ->String {
    return "Hello \(name)"
}
func hello6(name:String, f:(String)->String) -> String {
    return f(name)
}
hello6("Frank", sayHello)

//7.闭包
hello6("Frank", {(name:String)->String in
    return "Hello \(name)"
    })

//8.闭包简写，省略参数方法的参数类型
hello6("Frank", {name in
    return "Hello \(name)"
})

//9.闭包简写,省略参数方法的声明
hello6("Frank", {
    return "Hello \($0)"
})

{% endhighlight %}
