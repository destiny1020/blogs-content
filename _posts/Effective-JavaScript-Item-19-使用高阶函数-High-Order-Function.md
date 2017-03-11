---
title: '[Effective JavaScript] Item 19 使用高阶函数 (High-Order Function)'
date: 2014-09-15 09:54:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。
 
## 重点
 
不要被高阶函数这个名字给唬住了。实际上，高阶函数只是代表了两类函数：

1. 接受其他函数作为参数的函数
2. 返回值为函数的函数

有了这个定义，你也许就发现你已经使用过它们了，典型的就是对于一些事件的处理时传入的回调函数。
 
另外的一个典型使用场景就是Array类型的sort函数，它可以接受一个function作为排序时比较的判断依据：

```js
[3, 1, 4, 1, 5, 9].sort(function(x, y) {  
    if (x < y) {  
        return -1;  
    }  
    if (x > y) {  
        return 1;  
    }  
    return 0;  
}); // [1, 1, 3, 4, 5, 9]  
```

<!-- More -->

使用高阶函数能够代码更加清晰和整洁。在对于集合类型的数据进行操作时，可以考虑使用它们，比如可以将下面的for循环转换一下：

```js
var names = ["Fred", "Wilma", "Pebbles"];  
var upper = [];  
for (var i = 0, n = names.length; i < n; i++) {  
    upper[i] = names[i].toUpperCase();  
}  
upper; // ["FRED", "WILMA", "PEBBLES"]  
```

使用ES5中引入的map函数：

```js
var names = ["Fred", "Wilma", "Pebbles"];  
var upper = names.map(function(name) {  
    return name.toUpperCase();  
});  
  
upper; // ["FRED", "WILMA", "PEBBLES"]
```

当然，在非ES5兼容的浏览器中，如果也想使用map等高阶函数，可以使用underscore或者lodash。它们提供了非常多的对于Array，Object的操作。
 
当代码中出现了很多重复的片段时，就可以考虑利用高阶函数来对它们进行重构了。比如下面的代码中，出现了三段类似的代码，第一段用来生成字母表，第二段用来生成数字表，最后一段用来生成长度固定的随机字符串：

```js
var aIndex = "a".charCodeAt(0); // 97  
var alphabet = "";  
for (var i = 0; i < 26; i++) {  
    alphabet += String.fromCharCode(aIndex + i);  
}  
alphabet; // "abcdefghijklmnopqrstuvwxyz"  
  
var digits = "";  
for (var i = 0; i < 10; i++) {  
    digits += i;  
}  
digits; // "0123456789"  
  
var random = "";  
for (var i = 0; i < 8; i++) {  
    random += String.fromCharCode(Math.floor(Math.random() * 26)  
+ aIndex);  
}  
random; // "bdwvfrtp" (different result each time)  
```

可以将上面的三种情形中的不同部分封装到一个callback中，然后使用一个高阶函数来处理，高阶函数将三种情形中的公共部分抽取出来：

```js
function buildString(n, callback) {  
    var result = "";  
    for (var i = 0; i < n; i++) {  
        result += callback(i);  
    }  
    return result;  
}  
```

那么以上的三种情形就可以这样实现：

```js
var alphabet = buildString(26, function(i) {  
    return String.fromCharCode(aIndex + i);  
});  
alphabet; // "abcdefghijklmnopqrstuvwxyz"  
  
var digits = buildString(10, function(i) { return i; });  
digits; // "0123456789"  
  
var random = buildString(8, function() {  
    return String.fromCharCode(Math.floor(Math.random() * 26) + aIndex);  
});  
random; // "ltvisfjr" (different result each time)  
```

很显然，这种实现方式可以覆盖更多的可能性，因为变动的部分使用callback进行表示。同时，这种方式也符合编码的最佳实践，当循环部分需要变化的时候，你只需要修改一个地方就好了。
 
## 总结

1. 高阶函数就是讲其它函数作为参数或者返回函数作为返回值的函数
2. 学习使用map等方法，学习使用lodash或者underscore库
3. 发现代码中重复的代码片段，使用高阶函数对它们进行重构



