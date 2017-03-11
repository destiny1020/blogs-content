---
title: '[Effective JavaScript] Item 22 使用arguments来创建接受可变参数列表的函数'
date: 2014-09-18 10:08:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。

## 重点
 
在Item 21中，介绍了结合apply方法实现的可变参数列表函数average，它实际上只声明了一个数组作为参数，但是利用apply方法，实际上可以接受若干元素作为参数：

```js
function averageOfArray(a) {  
    for (var i = 0, sum = 0, n = a.length; i < n; i++) {  
        sum += a[i];  
    }  
    return sum / n;  
}  
averageOfArray.apply(null, [1, 2, 3, 4, 5]); 
```

<!-- More -->

而利用arguments变量，可以将声明的参数也去掉。即函数可以不显式声明任何参数。arguments对象提供了一个类似数组的使用方法：可以使用索引进行访问，并且它拥有length属性来表示其中含有多少个元素，所以，上面的函数可以这样实现：

```js
function average() {  
    for (var i = 0, sum = 0, n = arguments.length; i < n; i++) {  
        sum += arguments[i];  
    }  
    return sum / n;  
}  
```

以上的声明方式让average函数更加灵活，但是在处理数组参数时，需要结合apply方法，因为apply方法可以将数组参数“打散”成单个元素，然后这些元素又构成了arguments变量。
 
一个经验法则是：当你提供了利用arguments变量的函数的同时，也提供一个参数长度固定的版本，因为前者总是可以利用后者：

```js
function average() {  
    return averageOfArray(arguments);  
}  
```

这样的话，用户就可以在不使用apply方法的情况下，调用你提供的函数。因为在使用apply方法时，会损失一部分的代码可读性以及运行性能。
 
## 总结

1. 使用隐式的arguments对象来实现接受变长参数列表的函数
2. 在提供的变长参数函数的同时，也提供固定长度参数的函数，以避免使用apply方法
