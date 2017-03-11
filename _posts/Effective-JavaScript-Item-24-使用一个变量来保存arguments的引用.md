---
title: '[Effective JavaScript] Item 24 使用一个变量来保存arguments的引用'
date: 2014-09-19 19:49:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。
 
## 重点 
 
假设需要一个API用来遍历若干元素，像下面这样：

```js
var it = values(1, 4, 1, 4, 2, 1, 3, 5, 6);  
it.next(); // 1  
it.next(); // 4  
it.next(); // 1  
```

<!-- More -->

相应的实现可以是：

```js
function values() {  
    var i = 0, n = arguments.length;  
    return {  
        hasNext: function() {  
            return i < n;  
        },  
        next: function() {  
            if (i >= n) {  
                throw new Error("end of iteration");  
            }  
            return arguments[i++]; // wrong arguments  
        }  
    };  
}  
```

但是执行的实际情况却是：

```js
var it = values(1, 4, 1, 4, 2, 1, 3, 5, 6);  
it.next(); // undefined  
it.next(); // undefined  
it.next(); // undefined  
```

原因在于：对于arguments对象的赋值是隐式完成的。
在next方法内部，使用了arguments，然而此arguments和values方法开始处的arguments并不是一个对象。
 
解决方法也很简单，就是将需要访问的arguments使用另外一个变量进行引用。然后通过闭包的性质在其嵌套的函数中进行访问就可以了，像下面这样：

```js
function values() {  
    var i = 0, n = arguments.length, a = arguments;  
    return {  
        hasNext: function() {  
            return i < n;  
        },  
        next: function() {  
            if (i >= n) {  
                throw new Error("end of iteration");  
            }  
            return a[i++];  
        }  
    };  
}  
var it = values(1, 4, 1, 4, 2, 1, 3, 5, 6);  
it.next(); // 1  
it.next(); // 4  
it.next(); // 1  
```

## 总结

1. 当在嵌套的函数中使用arguments时，注意arguments的实际指向
2. 需要在嵌套的函数中使用外部函数的arguments时，将外部函数的arguments对象保存到一个变量中，让嵌套函数进行访问
