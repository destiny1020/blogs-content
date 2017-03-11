---
title: '[Effective JavaScript] Item 34 在prototype上保存方法'
date: 2014-10-08 17:06:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为EffectiveJavaScript的读书笔记。
 
## 重点 

不使用prototype进行JavaScript的编码是完全可行的，例如：

```js
function User(name, passwordHash) {  
    this.name = name;  
    this.passwordHash = passwordHash;  
    this.toString = function() {  
        return "[User " + this.name + "]";  
    };  
    this.checkPassword = function(password) {  
        return hash(password) === this.passwordHash;  
    };  
}  
  
var u1 = new User(/* ... */);  
var u2 = new User(/* ... */);  
var u3 = new User(/* ... */);  
```

<!-- More -->

当创建了多个User类型的实例时，就存在问题了：不仅是name和passwordHash属性在每个实例上都存在，toString和checkPassword方法在每个实例上都有一份拷贝。就像下图表示的那样：

![](http://img.blog.csdn.net/20141008170407734?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZG1fdmluY2VudA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

但是，当toString和checkPassword被定义在prototype上时，上图就变成下面这个样子了：

![](http://img.blog.csdn.net/20141008170534271?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZG1fdmluY2VudA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

toString和checkPassword方法现在定义在了User.prototype对象上，也就意味着这两个方法只存在一份拷贝，并被所有的User实例共享。
 
也许你会认为将方法作为拷贝放在每个实例上，会节省方法查询的时间。(当方法定义在prototype上时，首先会在实例本身上寻找方法，如果没有找到才会去prototype上继续找)
 
但是在现代的JavaScript执行引擎中，对方法的查询进行了大量优化，所以这个查询时间几乎是不需要考虑的，那么将方法放在prototype对象上就节省了很多内存。
 
## 总结

1. 将方法存放在实例上会导致每个实例都会拥有该方法的一份拷贝，导致内存的浪费。
2. 优先将方法存放在prototype对象上。
