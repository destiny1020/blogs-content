---
title: '[Effective JavaScript] Item 35 使用闭包来保存私有数据'
date: 2014-10-09 10:10:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为EffectiveJavaScript的读书笔记。
 
## 重点 

JavaScript的对象系统从其语法上而言并不鼓励使用信息隐藏(Information Hiding)。因为当使用诸如this.name，this.passwordHash的时候，这些属性默认的访问级别就是public的，在任何位置都能够通过obj.name，obj.passwordHash来对这些属性进行访问。

在ES5环境中，也提供了一些方法来更方便的访问一个对象上所有的属性，比如Object.keys()，Object.getOwnPropertyNames()。所以，一些开发人员使用一些规约来定义JavaScript对象的私有属性，比如最典型的是使用下划线作为属性的前缀来告诉其他开发人员和用户这个属性是不应该被直接访问的。

<!-- More -->
 
但是这样做，并不能从根本上解决问题。其他开发人员和用户还是能够对带有下划线的属性进行直接访问。对于确实需要私有属性的场合，可以使用闭包进行实现。
 
从某种意义而言，在JavaScript中，闭包对于变量的访问策略和对象的访问策略是两个极端。闭包中的任何变量默认都是私有的，只有在函数内部才能访问这些变量。比如，可以将User类型实现如下：

```js
function User(name, passwordHash) {  
    this.toString = function() {  
        return "[User " + name + "]";  
    };  
    this.checkPassword = function(password) {  
        return hash(password) === passwordHash;  
    };  
}  
```

此时，name和passwordHash都没有被保存为实例的属性，而是通过局部变量进行保存。然后根据闭包的访问规则，实例上的方法可以对它们进行访问，而在其它地方则不能。
 
使用这种模式的一个缺点是，利用了局部变量的方法都需要被定义在实例本身上，不能讲这些方法定义在prototype对象上。正如在Item34中讨论的那样，这样做的问题是会增加内存的消耗。但是在某些特别的场合下，即使将方法定义在实例上也是可行的。
 
## 总结

1. 闭包中定义的变量是私有的，只能在闭包中被引用。
2. 使用闭包来实现方法中的信息隐藏。
