---
title: '[Effective JavaScript] Item 29 避免使用非规范的Stack Inspection属性'
date: 2014-09-26 12:38:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。
 
## 重点 
 
由于历史原因，很多JavaScript执行环境中都提供了某些方式来查看函数调用栈。在一些环境中，arguments对象(关于该对象可以查看Item 22，23，24)上有两个额外的属性：
 
- arguments.callee - 它引用了正在被调用的函数
- arguments.caller - 它引用了调用当前函数的函数

关于arguments.callee的使用，可以参考下面的代码：

```js
var factorial = (function(n) {  
    return (n <= 1) ? 1 : (n * arguments.callee(n - 1));  
}); 
```

<!-- More -->

可见，在递归函数中，可以使用callee来得到当前正在被调用的函数。
 
但是，使用函数声明的方式也可以很方便的实现函数的递归调用，并且这种方式更加清晰：

```js
function factorial(n) {  
    return (n <= 1) ? 1 : (n * factorial(n - 1));  
}  
```

而对于arguments.caller，它提供的功能就更加强大了，能保存了调用当前函数的函数的一个引用。因为它有安全隐患，所以很多JavaScript运行环境都将这个属性移除了。同时，有部分运行环境在函数对象上提供了一个caller属性来达到和arguments.caller相同的效果：

```js
function revealCaller() {  
    return revealCaller.caller;  
}  
function start() {  
    return revealCaller();  
}  
start() === start; // true  
```

因此，可以利用这个属性来得到当前调用栈的信息：

```js
function getCallStack() {  
    var stack = [];  
    for (var f = getCallStack.caller; f; f = f.caller) {  
        stack.push(f);  
    }  
    return stack;  
}  
```

对于简单的调用关系，上述确实能够得到调用栈的信息：

```js
function f1() {  
    return getCallStack();  
}  
function f2() {  
    return f1();  
}  
var trace = f2();  
trace; // [f1, f2]  
```

但是当一个函数在调用栈中出现不止一次时，就会发生问题了，比如下面的代码会产生一个死循环：

```js
function f(n) {  
    return n === 0 ? getCallStack() : f(n - 1);  
}  
var trace = f(1); // infinite loop  
```

原因在于，当发生递归调用时，函数自身会被赋值给它的caller属性。因此getCallStack中的for循环的终止条件f永远不会为false：

```js
for (var f = getCallStack.caller; f; f = f.caller) {  
    stack.push(f);  
}  
```

正因为这种不稳定性和由此带来的安全性问题，在ES5的strict mode中，使用caller或者callee属性都是被禁止的：

```js
function f() {  
    "use strict";  
    return f.caller;  
}  
f(); // error: caller may not be accessed on strict functions  
```

## 总结

1. 避免使用arguments对象上的callee和caller属性
2. 避免使用function对象上的caller属性

