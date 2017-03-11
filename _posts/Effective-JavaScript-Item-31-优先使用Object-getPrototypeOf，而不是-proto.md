---
title: '[Effective JavaScript] Item 31 优先使用Object.getPrototypeOf，而不是__proto__'
date: 2014-09-30 10:08:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。
 
## 重点 
 
在ES5中引入了Object.getPrototypeOf作为获取对象原型对象的标准API。但是在很多执行环境中，也提供了一个特殊的\_\_proto\_\_属性来达到同样的目的。
 
因为并不是所有的环境都提供了这个\_\_proto\_\_属性，且每个环境的实现方式各不相同，因此一些结果可能不一致：

```js
// 在某些环境中  
var empty = Object.create(null); // object with no prototype  
"__proto__" in empty; // false (in some environments)  
  
// 在某些环境中  
var empty = Object.create(null); // object with no prototype  
"__proto__" in empty; // true (in some environments) 
```

<!-- More -->

所以当环境中支持Object.getPrototypeOf方法时，优先使用它。即使不支持，也可以为了实现一个：

```js
if (typeof Object.getPrototypeOf === "undefined") {  
    Object.getPrototypeOf = function(obj) {  
        var t = typeof obj;  
        if (!obj || (t !== "object" && t !== "function")) {  
            throw new TypeError("not an object");  
        }  
        return obj.__proto__;  
    };  
}  
```

上述代码首先会对当前环境进行检查，如果已经支持了Object.getPrototypeOf，就不会再重复定义。
 
另外，在使用\_\_proto\_\_时会导致一些错误，在Item 45中会进行讨论。
 
## 总结

1. 优先使用标准方法Object.getPrototypeOf，而不是非标准的\_\_proto\_\_属性。
2. 为非ES5环境实现一个Object.getPrototypeOf方法，从而保持代码的一致性。



