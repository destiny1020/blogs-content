---
title: '[Effective JavaScript] Item 37 认识this的隐式指向'
date: 2014-10-11 10:17:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。

## 重点
 
CSV数据通常都会被某种分隔符进行分隔，所以在实现CSV Reader时，需要支持不同的分隔符。那么，很自然的一种实现就是将分隔符作为构造函数的参数。

```js
function CSVReader(separators) {  
    this.separators = separators || [","];  
    this.regexp =  
        new RegExp(this.separators.map(function(sep) {  
            return "\\" + sep[0];  
        }).join("|"));  
}  
```

<!-- More -->

对于CSV Reader而言，它的工作原理是首先将读入的字符串根据换行符切分成为一个个的行，然后对每行根据分隔符进行切分成为一个个的单元。所以，可以使用map方法进行实现：

```js
CSVReader.prototype.read = function(str) {  
    var lines = str.trim().split(/\n/);  
    return lines.map(function(line) {  
        return line.split(this.regexp); // wrong this!  
    });  
};  
var reader = new CSVReader();  
reader.read("a,b,c\nd,e,f\n"); // [["a,b,c"], ["d,e,f"]], wrong result 
```

可是上述代码中有一个错误：传入到map函数中的回调函数的this指向有问题。即其中的this.regexp并不能正确的引用到CSVReader实例的regexp属性。因此，最后得到的结果也就是不正确的了。
 
对于这个例子，在map的回调函数中this指向的实际上是全局对象window。关于this在各种场景下的指向，在Item 18和Item 25中进行了介绍。
 
为了克服this的指向问题，map函数提供了第二个参数用来指定在其回调函数中this的指向：

```js
CSVReader.prototype.read = function(str) {  
    var lines = str.trim().split(/\n/);  
    return lines.map(function(line) {  
        return line.split(this.regexp);  
    }, this); // forward outer this-binding to callback  
};  
var reader = new CSVReader();  
reader.read("a,b,c\nd,e,f\n");  
// [["a","b","c"], ["d","e","f"]]  
```

但是，并不是所有的函数都如map考虑的这么周全。如果map函数不能接受第二个参数作为this的指向，可以使用下面的方法：

```js
CSVReader.prototype.read = function(str) {  
    var lines = str.trim().split(/\n/);  
    var self = this; // save a reference to outer this-binding  
    return lines.map(function(line) {  
        return line.split(self.regexp); // use outer this  
    });  
};  
var reader = new CSVReader();  
reader.read("a,b,c\nd,e,f\n");  
// [["a","b","c"], ["d","e","f"]]  
```

这种方法将this的引用保存到了另外一个变量中，然后利用闭包的特性在map的回调函数中对它进行访问。通常会使用变量名self来保存this的引用，当然使用诸如me，that也是可行的。
 
在ES5环境中，还可以借助于函数的bind方法来绑定this的指向(在Item 25中，对该方法进行了介绍)：

```js
CSVReader.prototype.read = function(str) {  
    var lines = str.trim().split(/\n/);  
    return lines.map(function(line) {  
        return line.split(this.regexp);  
    }.bind(this)); // bind to outer this-binding  
};  
var reader = new CSVReader();  
reader.read("a,b,c\nd,e,f\n");  
// [["a","b","c"], ["d","e","f"]]  
```

## 总结

1. 根据函数的调用方式的不同，this的指向也会不同。
2. 使用self，me，that来保存当前this的引用供其他函数使用。