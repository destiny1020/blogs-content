---
title: '[Effective JavaScript] Item 36 实例状态只保存在实例对象上'
date: 2014-10-10 10:30:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为EffectiveJavaScript的读书笔记。
 
## 重点  

一个类型的prototype和该类型的实例之间是”一对多“的关系。那么，需要确保实例相关的数据不会被错误地保存在prototype之上。比如，对于一个实现了树结构的类型而言，将它的子节点保存在该类型的prototype上就是不正确的：

```js
function Tree(x) {  
    this.value = x;  
}  
Tree.prototype = {  
    children: [], // should be instance state!  
    addChild: function(x) {  
        this.children.push(x);  
    }  
};  
  
var left = new Tree(2);  
left.addChild(1);  
left.addChild(3);  
  
var right = new Tree(6);  
right.addChild(5);  
right.addChild(7);  
  
var top = new Tree(4);  
top.addChild(left);  
top.addChild(right);  
  
top.children; // [1, 3, 5, 7, left, right]  
```

<!-- More -->

当状态被保存到了prototype上时，所有实例的状态都会被集中地保存，在上面这种场景中显然是不正确的：本来属于每个实例的状态被错误地共享了。如下图所示：

![](http://img.blog.csdn.net/20141010103028475?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZG1fdmluY2VudA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

正确的实现应该是这样的：

```js
function Tree(x) {  
    this.value = x;  
    this.children = []; // instance state  
}  
Tree.prototype = {  
    addChild: function(x) {  
        this.children.push(x);  
    }  
};  
```

此时，实例状态的存储如下所示：

![](http://img.blog.csdn.net/20141010103033389?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZG1fdmluY2VudA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

可见，当本属于实例的状态被共享到prototype上时，也许会产生问题。在需要在prototype上保存状态属性前，一定要确保该属性是能够被共享的。
 
总体而言，当一个属性是不可变(无状态)的属性时，就能将它保存在prototype对象上(比如方法能够被保存在prototype对象上就是因为这一点)。当然，有状态的属性也能够被放在prototype对象上，这要取决于具体的应用场景，典型的比如用来记录一个类型实例数量的变量。使用Java语言作为类比的话，这类能够存储在prototype对象上的变量就是Java中的类变量(使用static关键字修饰)。
 
## 总结

1. 当在prototype对象上存放可变数据时，可能会带来问题。
2. 一般情况下，每个实例特有的可变属性需要被存放在实例本身上。




