---
layout: post
title: "同学，你的二叉树【掉了/找到了】"
description: "二叉树翻转实现"
header_image: http://7u2jir.com1.z0.glb.clouddn.com/img/2018-03-06-01.png
keywords: "二叉树翻转"
tags: [算法]
---
{% include JB/setup %}
![img](http://7u2jir.com1.z0.glb.clouddn.com/img/2018-03-06-01.png)

## 前言

> 什么是大神？

每个人的评判标准不同，但是大神一定有过人之处，比如工具写的溜，算法牛逼，编码功力强，等等。

大概15年的时候出过一档事，`Homebrew`的作者面试Google被刷了，其中涉及了移到算法题。具体内容已经不记得，
今天碰巧遇到一个二叉树的的问题，Google了一把相关算法。神迹出现了，正是当年`Homebrew`大神Fuck off的`二叉树翻转`。

迷之尴尬，这大概是我离`大神`最近的一次了。

当然这是玩笑话了，下面我们一起看看”二叉树翻转“的几种实现方案。

## 二叉树翻转

二叉树的概念就不介绍了，如果不知道是什么可以好好补习一下数据结构，再看下文。

首先讲一下翻转的情况，主要做左右翻转，如图：

```
  1         1
 / \       / \
22  33 => 33  22
   /       \
  444      444
```

## 递归版本

笔者第一反应是，用数组构造二叉树，然后遍历节点，同时左右节点替换。
但实际上最简单的写法根本不需要使用数组来表示二叉树，思路如下：

1. 定义节点;
2. 从根节点开始，左右节点就交换；
3. 并对交换后的左右节点递归调用自身；
4. 注意合法性判断，比如，二叉树并不是完美二叉树，怎么处理左右节点缺失的情况；

下面是实现代码：

```java
// define tree node
class Node {
  Node left;
  Node right;
  String value;//值可以随意
  Node(String value) {
    this.value = value;
  }
}

// define reverse method 
public static void invert(Node node) {
  if (node == null) {
    // skip inlvaid node
    return;
  }

  Node temp = node.left;
  node.left = node.right;
  node.right = temp;
  // left node
  invert(node.left);
  // right node
  invert(node.right);
}

// prepare any nodes
public static Node mockNodes(){
  Node one = new Node("1");
  one.left = new Node("22");
  one.right = new Node("33");
  one.right.left = new Node("444");
  return one;
}

// reverse nodes
public static void main(String... args) {
  Node root = mockNodes();
  invert(root);
}
```
在上面的实现代码中，为了更加完整，构造了一个测试用的二叉树，然后调用我们的翻转函数，递归处理左右节点。

## 非递归版本

递归写起来好理解，但是会造成方法栈的嵌套层数递增，我们知道递归算法往往还会存在一个非递归算法，通过引入一个中间量实现层级化解问题。

下面来看一下非递归的思路：

1. 首先任然是定义Node节点；
2. 其次选择一个具备入栈出栈的数据结构，比如LinkedList，Vector，甚至ArrayList也是可以的；
3. 从根节点开始，将Node放入列表中；
4. 列表/链表的作用是暂存需要翻转的节点，因此没反转一个节点，即将他的潜在左右子节点加入列表/链表；知道列表/链表为空；

实现代码：

```java
// define tree node
class Node {
  Node left;
  Node right;
  String value;//值可以随意
  Node(String value) {
    this.value = value;
  }
}

// define reverse method 
public static void invertWithoutRecurring(Node node) {
  if (node == null) {
    return;
  }
  // middle strcut to hold nodes which need to be invert
  LinkedList<Node> stack = new LinkedList<Node>();
  // start from root node
  stack.push(node);
  while(!stack.isEmpty()) {
    //
    Node n = stack.pop();
    // swap left and right sub-nodes
    Node temp = n.left;
    n.left = n.right;
    n.right = temp;
    // push sub-nodes if exist
    if(n.left != null) {
      stack.push(n.left);
    }
    if(n.right != null) {
      stack.push(n.right);
    }
  }
}

// prepare any nodes
public static Node mockNodes(){
  Node one = new Node("1");
  one.left = new Node("22");
  one.right = new Node("33");
  one.right.left = new Node("444");
  return one;
}

// reverse nodes
public static void main(String... args) {
  Node root = mockNodes();
  invertWithoutRecurring(root);
}

```

细心地你可能会考虑，既然引入中间数据结构只是为了保存待翻转的Node节点，从实现上来说，并没有利用到LinkedList之类过多属性，是不可以换轻量级的数据来实现？

当然是可以的，下面把翻转部分换成数组实现。

思路：

1. 主体和前面的非递归一致，区别在于，我们不适用任何预定义的数据结构，而是采用一个长度为3的数组；
2. 为数组增加isEmpty,pop,push三个方法，方便书写调用；
3. 主体替换LinkedList

```java
private void invertWithoutRecurring2(Node root) {
    if(root == null) {
        return;
    }
    // simple array with length to be 3 is enough for invert
    Node[] nodesTobeInvert = {root, null, null};
    while(!isEmpty(nodesTobeInvert)) {
      Node node = pop(nodesTobeInvert);
      
      Node temp = node.left;
      node.left = node.right;
      node.right = temp;
      
      push(nodesTobeInvert, node.left);
      push(nodesTobeInvert, node.right);
    }
}

private boolean isEmpty(Node[] nodes) {
  if(nodes == null) {
    return true;
  }
  int size = nodes.length;
  for(int i = 0; i< size; i++) {
    if(nodes[i] != null) {
      return false;
    }
  }
  return true;
}
private Node pop(Node[] nodes){
  if(nodes == null) {
    return null;
  }
  int size = nodes.length;
  for(int i = 0; i< size; i++) {
    if(nodes[i] != null) {
      Node temp = nodes[i];
      nodes[i] = null;
      return temp;
    }
  }
  return null;
}

private void push(Node[] nodes, Node node) {
  if(node == null || nodes == null) {
    return;
  }
  int size = nodes.length;
  for(int i = 0; i< size; i++) {
    if(nodes[i] == null) {
      nodes[i] = node;
      return;
    }
  }
}
```

## 数组表示二叉树

> 是不是可以用数组表示二叉树，进而反转二叉树？

实际上这是一个看似OK的问题，由于二叉树存在各种子节点为空的情况，因此直接用数组来表示二叉树，必然会引起大量为空的节点表示（为空节点不可以省去）。

从编码的直观性和编码效率来看，用数组来处理二叉树的翻转是不太明智的做法。


```
  1         1
 / \       / \
22  33 => 33  22
   /       \
  444      444
```

上述二叉树用数组表示为：`{"1","22","33",null,null,"444",null}`,  翻转之后为：`{"1", "33", "22", null, "444", null, null}`

我们用代码构造

```java
// prepare any nodes
public static String[] mockNodes(){
  String[] nodes = new String()[2^0+2^1+2^2];
  nodes[0] = "1";
  int index = leftIndex(0);
  nodes[index] = "22"
  nodes[index+1] = "33";
  nodes[leftIndex(index)] = null;
  nodes[rightIndex(index)] = null;
  nodes[leftIndex(index+1)] = "4444";
  nodes[rightIndex(index+1)] = null;
  return nodes;
}

private int leftIndex(int curentIndex) {
  return 2 * currentIndex + 1;
}

private int rightIndex(int currentIndex) {
  return 2 * currentIndex + 2;
}
```
如果翻转数组形式的二叉树？未完待续