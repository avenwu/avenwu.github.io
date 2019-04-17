---
layout: post
title: "[swift]不一样的switch/case"
header_image: /assets/img/2016-03-06-21.jpg
description: ""
category: "swift"
tags: [swift]
---
{% include JB/setup %}
![img](/assets/img/2016-03-06-21.jpg)


在java中用switch有不少限制，只能用常量表达式，case必须是常量，如果用枚举类型还要考虑jdk版本。

swift中switch/case显得强大一些特别是case中支持where语句

{% highlight swift %}

//switch示例
enum Direction: String {
    case EAST = "东"
    case SOUTH = "南"
    case WEST = "西"
    case NORTH = "北"
}

struct Path {
    var direction : Direction
    var miles : Int
}
//枚举可以省略枚举类型，直接写具体值
let path1 = Path(direction: .EAST, miles: 100)
let path2 = Path(direction: Direction.WEST, miles: 200)

//不需要break，默认是不管穿的
func printPath(path : Path) {
    switch path.direction {
    case Direction.EAST:
        println("向东\(path.miles)")
    case .WEST where path.miles > 150:
        println("向西超出150范围")
    case .WEST:
        println("向西\(path.miles)")
    default://默认是必须的，否则提示错误警告
        println("方向：\(path.direction), 距离：\(path.miles)")
    }
}

printPath(path1)//向东100
printPath(path2)//向西超出150范围
printPath(Path(direction: .WEST, miles: 90))//向西90

{% endhighlight %}

上面的case如果用java来写，那么WEST的两个判断需要合并在一起，通过if/else处理。这里的枚举声明页比较简洁，我们直接继承String，不需要显示声明一个额外的成员来保存枚举字符的值。


---
