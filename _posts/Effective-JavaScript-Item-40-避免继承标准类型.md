---
title: '[Effective JavaScript] Item 40 避免继承标准类型'
date: 2014-10-15 09:59:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。
 
## 重点 
 
ECMAScript标准库不大，但是提供了一些重要的类型如Array，Function和Date。在一些场合下，你也许会考虑继承其中的某个类型来实现特定的功能，但是这种做法并不被鼓励。
 
比如为了操作一个目录，可以让目录类型继承Array类型如下：

```js
function Dir(path, entries) {  
    this.path = path;  
    for (var i = 0, n = entries.length; i < n; i++) {  
        this[i] = entries[i];  
    }  
}  
Dir.prototype = Object.create(Array.prototype);  
// extends Array  
  
var dir = new Dir("/tmp/mysite", ["index.html", "script.js", "style.css"]);  
dir.length; // 0  
```

<!-- More -->

但是可以发现，dir.length的值是0，而不是期待中的3。
 
发生这种现象的原因在于：只有当对象是真正的Array类型时，length属性才会起作用。
 
在ECMAScript标准中，定义了一个不可见的内部属性被称为 [[class]]。该属性的值只是一个字符串，所以不要被误导认为JavaScript也实现了自己的类型系统。所以，对于Array类型，这个属性的值就是“Array”；对于Function类型，这个属性的值就是“Function”。下表是ECMAScript定义的所有[[class]] 值：

![](http://img.blog.csdn.net/20141015095438015?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZG1fdmluY2VudA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

那么当对象的类型确实是Array时，length属性的特别之处就在于：length的值会和该对象中被索引的属性个数保持一致。比如对于一个数组对象arr，arr[0]和arr[1]就表示该对象有两个被索引的属性，那么length的值就是2。当添加了arr[2]的时候，length的值会被自动同步成3。同样地，当设置length值为2时，arr[2]会被自动设置成undefined。
 
但是当继承Array类型并创建实例时，该实例的 [[class]] 属性并不是Array，而是Object。因此length属性不能正确的工作。
 
在JavaScript中，也提供了用于查询 [[class]] 属性的方法，即使用Object.prototype.toString方法：

```js
var dir = new Dir("/", []);  
Object.prototype.toString.call(dir); // "[object Object]"  
Object.prototype.toString.call([]); // "[object Array]"  
```

因此，更好的实现方法是使用组合而不是继承：

```js
function Dir(path, entries) {  
    this.path = path;  
    this.entries = entries; // array property  
}  
Dir.prototype.forEach = function(f, thisArg) {  
    if (typeof thisArg === "undefined") {  
        thisArg = this;  
    }  
    this.entries.forEach(f, thisArg);  
};  
```

以上代码将不再使用继承，而是将一部分功能代理给内部的entries属性来实现，该属性的值是一个Array类型对象。
 
ECMAScript标准库中，大部分的构造函数都会依赖内部属性值如 [[class]] 来实现正确的行为。对于继承这些标准类型的子类型，无法保证它们的行为是正确的。因此，不要继承ECMAScript标准库中的类型如：
Array， Boolean， Date， Function， Number，RegExp，String
 
## 总结

1. 继承标准类型可能会导致子类的行为不正确，因为标准类型会依赖于内部属性诸如 [[class]]
2. 优先使用组合的方式来实现功能，而不是使用继承。
