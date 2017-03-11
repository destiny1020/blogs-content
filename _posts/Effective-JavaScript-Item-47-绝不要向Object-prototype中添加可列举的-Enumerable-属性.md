---
title: '[Effective JavaScript] Item 47 绝不要向Object.prototype中添加可列举的(Enumerable)属性'
date: 2014-11-10 10:02:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。
 
## 重点 
 
如果你的代码中依赖于for..in循环来遍历Object类型中的属性的话，不要向Object.prototype中添加任何可列举的属性。
 
但是在对JavaScript执行环境进行增强的时候，往往都需要向Object.prototype对象添加新的属性或者方法。比如可以添加一个方法用于得到某个对象中的所有的属性名：

```js
Object.prototype.allKeys = function() {  
    var result = [];  
    for (var key in this) {  
        result.push(key);  
    }  
    return result;  
}; 
```

<!-- More -->

但是结果是下面这个样子的：
 
```js
({ a: 1, b: 2, c: 3}).allKeys(); // ["allKeys", "a", "b","c"]
```
 
一个可行的解决方案是使用函数而不是在Object.prototype上定义新的方法：

```js
function allKeys(obj) {  
    var result = [];  
    for (var key in obj) {  
        result.push(key);  
    }  
    return result;  
}  
```

但是如果你确实需要向Object.prototype上添加新的属性，同时也不希望该属性在for..in循环中被遍历到，那么可以利用ES5环境提供的Object.defineProject方法：

```js
Object.defineProperty(Object.prototype, "allKeys", {  
    value: function() {  
        var result = [];  
        for (var key in this) {  
            result.push(key);  
        }  
        return result;  
    },  
    writable: true,  
    enumerable: false,  
    configurable: true  
});  
```

以上代码的关键部分就是将enumerable属性设置为false。这样的话，在for..in循环中就无法遍历该属性了。
 
## 总结

1. 避免向Object.prototype中添加任何属性。
2. 如果确实有必要向Object.prototype中添加方法属性，可以考虑使用独立函数替代。
3. 使用Object.defineProperty来添加可以不被for..in循环遍历到的属性。

