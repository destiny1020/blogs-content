---
title: '[Effective JavaScript] Item 13 使用即时调用的函数表达式(IIFE)来创建局部域'
date: 2014-09-10 19:26:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。
 
## 重点 

所谓的即时调用的函数表达式，这个翻译也许不太准确，它对应的英文原文是Immediately Invoked Function Expression (IIFE)。下文也使用IIFE来表达这一概念。
 
首先看一个程序：

```js
function wrapElements(a) {  
    var result = [], i, n;  
    for (i = 0, n = a.length; i < n; i++) {  
        result[i] = function() { return a[i]; };  
    }  
    return result;  
}  
var wrapped = wrapElements([10, 20, 30, 40, 50]);  
var f = wrapped[0];  
f(); // ?  
```

<!-- More -->

这个程序的作用是，将传入到wrapElements中的数组的每个元素进行一次"打包"操作，将元素替换成一个返回它们自身的函数，
 
也许你会认为最后输出的而结果是10，但是最后的结果实际上是undefined。
 
会做出这种符合直觉的假设的原因是，在function() { return a[i]; }; 这段代码中，一般会认为这里的i就是当前循环时使用到的变量i，那么下面的赋值就应该成立：

```js
result[1] = function() { return a[1]; };  
```

但是，不要忘了一个重要的事实，在以上的赋值语句中，首先会创建一个闭包。在这个闭包中引用到了其外部的变量i，回顾在Item 11我们学习到的关于闭包的一个原则，就是闭包会以引用的形式存储其外部的变量，而不是以值的形式。
 
所以，在循环结束之后，i的值应该是5。那么在调用wrapped[0]()时，就相当于调用：return a[5]。很显然，这个值是不存在的，故返回undefined。
 
如果将上述代码换成下面这样：

```js
function wrapElements(a) {  
    var result = [];  
    for (var i = 0, n = a.length; i < n; i++) {  
        result[i] = function() { return a[i]; };  
    }  
    return result;  
}  
var wrapped = wrapElements([10, 20, 30, 40, 50]);  
var f = wrapped[0];  
f(); // ?  
```

可以发现，变量i和n直到for循环开始的时候才会开始。那么结果是怎么样的呢？
 
结果还是undefined。
 
这是因为VariableHoisting(Item 12)的缘故。无论在一个function中的哪里声明变量，该变量的定义部分总会出现在function的开始处。
 
那么，如何让程序以我们期待的方式运行呢，IIFE就派上用场了：

```js
function wrapElements(a) {  
    var result = [];  
    for (var i = 0, n = a.length; i < n; i++) {  
        (function() {  
            var j = i;  
            result[i] = function() { return a[j]; };  
        })();  
    }  
    return result;  
}  
```

在for循环中，创建了一个IIFE，将当前的循环变量i赋值给了j，然后在后面的function中返回的是a[j]。这相当使用了另外一个局部变量将外部变量当前的值给记录下来，以便将来使用。那么IIFE的价值就在于它能够克服JavaScript语言中没有块作用域(Block Scoping)的弊端：通过创建一个匿名的闭包，将某个外部变量的当前值给保存起来。就好比将一个变量的当前值"冻结"住，供将来使用。
 
IIFE的另一种形式是将需要"冻结"的变量作为参数传入到IIFE中：

```js
function wrapElements(a) {  
    var result = [];  
    for (var i = 0, n = a.length; i < n; i++) {  
        (function(j) {  
            result[i] = function() { return a[j]; };  
        })(i);  
    }  
    return result;  
}  
```

在使用IIFE的时候，有两个注意事项：

1. 如果在for或者while循环中使用了IIFE，那么注意不要在IIFE中包含break以及continue语句。因为break和continue只有在循环中才能够使用，否则抛出SyntaxError: Illegal break statement。
2. 如果IIFE中使用了this或者arguments，那么它们的意义可能会和你想象的有出入，关于这一点，在后续的Items中会进行介绍。

## 总结

1. 闭包会存储外部变量的引用，而不是值
2. 使用IIFE来创建局部作用域
