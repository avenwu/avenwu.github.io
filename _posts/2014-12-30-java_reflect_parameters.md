---
layout: post
title: "Java反射参数为数组"
header_image: /assets/img/2016-03-06-36.jpg
description: ""
category: 
tags: [android]
---
{% include JB/setup %}
![img](/assets/img/2016-03-06-36.jpg)

使用反射调用非公开的方法有时能解决许多问题，如果方法的参数是数组时类型该怎么传递呢？这里找到了一种方法记录一下

### 实例
比如：

{% highlight java %}
	
class A{
	private void sayHello(String[] names){
		//...
		System.out.println("sayHello invoked");
	} 
}
String[] names = new String[]{"A", "B", "C"};
Method sayHello = A.class.getDeclaredMethod("sayHello", String[].class);
sayHello.setAcess(true);
sayHello.invoke(new A(), new Object[]{names});

{% endhighlight %}

这里有两个地方需要注意

* A.class.getDeclaredMethod时后面的参数是数组，用加[];
* sayHello.invoke调用时直接传一个String[]实例会报异常，需要再次用Object[]包装一下；  

异常，比较奇怪,google后找到上面的解决方法：

{% highlight java %}

java.lang.IllegalArgumentException: argument 1 should have type java.lang.String[], got java.lang.String
at java.lang.reflect.Method.invokeNative(Native Method)
at java.lang.reflect.Method.invoke(Method.java:515)

{% endhighlight %}
