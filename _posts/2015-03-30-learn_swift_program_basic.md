---
layout: post
title: "swift里的struct与class"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-23.jpg
description: "swift语法基础"
category: "swift"
tags: [swift]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2016-03-06-23.jpg)

	在swift中结构体和类非常相似，两者都可以有各自的方法和成员，但是结构体实际上是值类型，且不能继承，和引用类型的类是不同的。

## struct值类型
通过一个实例可以清楚的体现出来。
定义一个结构体MyStruct,添加一个int型成员money,下面分别再两种情况下修改money的值，观察money的变化：


{% highlight swift %}

struct MyStruct {
    var money = 100 as Int
    
    mutating func  addMoney(count : Int){
        money += count;
    }
}
func changeMoney(var myStruct: MyStruct, addMore count : Int){
    myStruct.addMoney(count)
    println("Internal money = \(myStruct.money)")
}
var myStruct = MyStruct()
//通过changeMoney增加money，内部输出110
changeMoney(myStruct, addMore: 10)
//这里money还是100
println("Outside money = \(myStruct.money)")
//直接增加money
myStruct.addMoney(10)
println("Outside money = \(myStruct.money)")

{% endhighlight %}
![snapshot](http://7u2jir.com1.z0.glb.clouddn.com/swift_struct_1.png)


很明显由于strcut是值类型，调用方法changeMoney时实际上将myStruct拷贝了一份新的实例，所以在changeMoney方法内修改的并不是外面的myStruct。

## class引用类型
类的话没什么特殊和java中是相似的的属于引用类型：

{% highlight swift %}

class MyClass {
    var money = 100 as Int
    
    func addMonet(count : Int){
        money += count;
    }
}
func changeMoney(myClass : MyClass, addMore count : Int){
    myClass.addMonet(count)
    println("Internal money = \(myClass.money)")
}
var myClass = MyClass()
changeMoney(myClass, addMore: 10)

myClass.money
{% endhighlight %}

---
