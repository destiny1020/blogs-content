---
title: '[Effective JavaScript] Item 27 使用闭包而不是字符串来封装代码'
date: 2014-09-24 09:59:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。
 
对于代码封装，在JavaScript中有两种方式可以办到。第一种就是使用function，第二种则是利用eval()函数，传入到该函数的字符串参数可以是一段代码。
 
当对使用哪种方式犹豫不决时，使用function。因为使用字符串的一个重要缺点是，传入的字符串并不是一个闭包，而function则可以代表一个闭包。关于闭包的特点，在Item 11中进行了描述。
 
下面是一段使用字符串来封装代码的例子：

```js
function repeat(n, action) {  
    for (var i = 0; i < n; i++) {  
        eval(action);  
    }  
}  
```

<!-- More -->

传入到eval的action是一个字符串，用来代表需要被执行的逻辑，该字符串中出现的变量都会被解释成全局变量，在多数环境下，即window对象上的变量。
 
比如在下面的例子中：

```js
var start = [], end = [], timings = [];  
repeat(1000, "start.push(Date.now()); f(); end.push(Date.now())");  
for (var i = 0, n = start.length; i < n; i++) {  
    timings[i] = end[i] - start[i];  
}  
```

repeat函数的第二个参数是一段代码，其中用到了两个变量start和end。
它们引用的是全局变量start和end。当上面的代码不在任何函数中时，还能够正常工作，如果将它们放到了函数中：

```js
function benchmark() {  
    var start = [], end = [], timings = [];  
    repeat(1000, "start.push(Date.now()); f(); end.push(Date.now())");  
    for (var i = 0, n = start.length; i < n; i++) {  
        timings[i] = end[i] - start[i];  
    }  
    return timings;  
}  
```

此时的start和end变量仍然引用的是全局变量start和end。如果在运行时没有这两个全局变量的任意一个，就会报出ReferenceError异常，这还是最好的结果。如果不凑巧在全局对象中拥有这两个同名的属性，那么程序的运行结果就不可预测了。
 
更好的封装代码的方式是使用function：

```js
function repeat(n, action) {  
    for (var i = 0; i < n; i++) {  
        action();  
    }  
}  
```

此时，可以将上面的函数应用在任何函数的内部：

```js
function benchmark() {  
    var start = [], end = [], timings = [];  
    repeat(1000, function() {  
        start.push(Date.now());  
        f();  
        end.push(Date.now());  
    });  
    for (var i = 0, n = start.length; i < n; i++) {  
        timings[i] = end[i] - start[i];  
    }  
    return timings;  
}  
```

start和end能够引用到benchmark函数内部定义的start和end数组。这得益于闭包的性质。
 
另外一个不使用eval的原因是：在JavaScript的执行引擎中，对于以字符串形式封装的代码往往很难优化。因为字符串也许是动态生成的，编译器/解释器无法在执行前得知它的具体信息，就无法对它们进行优化。
 
##总结

1. 如果确实需要使用字符串来封装一段代码时，确保在其中不要依赖任何非全局变量
2. 优先使用闭包进行代码封装而不是使用字符串
