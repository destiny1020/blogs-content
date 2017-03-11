---
title: '[Effective JavaScript] Item 25 使用bind方法来得到一个固定了this指向的方法'
date: 2014-09-22 10:04:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。
 
## 重点 

当需要将方法抽取出来作为回调函数使用的时候，常常会因为this的指向不明而发生错误，比如：

```js
var buffer = {  
    entries: [],  
    add: function(s) {  
        this.entries.push(s);  
    },  
    concat: function() {  
        return this.entries.join("");  
    }  
};  
```

<!-- More -->

如果想利用其中的add作为回调函数对一组数据进行添加：

```js
var source = ["867", "-", "5309"];  
source.forEach(buffer.add); // error: entries is undefined  
```

以上的forEach方法时ES5中引入的，如果不是使用的ES5环境，可以使用lodash或者underscore库中的同名方法。
 
在Item 18中我们知道了当调用一个函数的时候，函数体中this的指向是由该函数的调用方式决定的，比如当这样调用时：buffer.add(something)，this的指向就是buffer对象。
 
然而当add方法作为回调函数在forEach中使用时，仅仅是将该它单纯地作为函数进行调用，所以this指向的是window或者undefined(根据是否使用strict mode而不同)。
 
此时，需要将this的指向也作为第二个参数传入到forEach中：

```js
var source = ["867", "-", "5309"];  
source.forEach(buffer.add, buffer);  
buffer.join(); // "867-5309"  
```

但并不是所有的高阶函数都像forEach这样能够让你传入一个this的指向，此时可以采用这种方式：

```js
var source = ["867", "-", "5309"];  
source.forEach(function(s) {  
    buffer.add(s);  
});  
buffer.join(); // "867-5309"  
```

上述代码创建了一个匿名函数，将add作为方法调用，就保证了this的指向是buffer对象。
 
因为在某些时候，函数中this的指向是固定的，所以为了支持这种场景，ES5中添加了bind方法：

```js
var source = ["867", "-", "5309"];  
source.forEach(buffer.add.bind(buffer));  
buffer.join(); // "867-5309"
```

它的实现原理就是上面使用匿名函数的方式，bind方法本身是一个高阶函数，它会根据传入的对象，返回另一个函数来当做回调函数(也许是这样的，原文中并没有讲述bind的实现，如有不妥，希望能够为我指正)：

```js
function bind(receiver) {  
    var fn = this;  
    return function(s) {  
        receiver.fn(s);  
    };  
}  
```

所以，bind方法本身不会修改被"装饰"的方法，而是在生成了一个新的方法用来完成this的指向。可以通过下面的代码来验证这一点：

```js
buffer.add === buffer.add.bind(buffer); // false  
```

这也意味着，在任何函数上使用bind都是安全的，比如被共享的函数和原型继承链上的函数，它不会修改原函数的行为，而是生成了新的函数。
 
## 总结

1. 注意当抽取其他对象的方法并调用的时候，this的指向也许不正确
2. 当将一个对象的方法传入到高阶函数中时，使用匿名函数来保证在调用该方法时，this的指向正确
3. 使用bind方法来“修饰”一个方法来确保this的指向是正确的
