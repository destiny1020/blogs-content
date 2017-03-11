---
title: '[Effective JavaScript] Item 26 使用bind来进行函数的柯里化(Curry)'
date: 2014-09-23 10:51:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。
 
在上一个Item中介绍了bind的一种用法：用来绑定this对象。但是实际上，bind含有另一种用法，就是帮助函数进行柯里化。关于柯里化，这里有一份[百科](http://zh.wikipedia.org/wiki/%E6%9F%AF%E9%87%8C%E5%8C%96)可以参考。

但是实际上，关于柯里化只需要记住一点就够了：柯里化是把接受多个参数的函数变换成接受一个单一参数(通常是最初函数的第一个参数，但是并无限制)的函数，并且返回这个接受单一参数函数的过程。
 
一个用来连接字符串得到URL的例子：

```js
function simpleURL(protocol, domain, path) {  
    return protocol + "://" + domain + "/" + path;  
}  
```

<!-- More -->

那么当有一系列的path需要被映射得到对应的URL的时候，可以借助ES5的map方法：

```js
var urls = paths.map(function(path) {  
    return simpleURL("http", siteDomain, path);  
});  
```

上面的方法实现起来不难，但是有提高的空间。注意到在调用simpleURL的时候，传入的前两个参数都是固定的，只有第三个参数path在每次调用时不一样。这时候bind就可以派上用场了：

```js
var urls = paths.map(simpleURL.bind(null, "http", siteDomain)); 
```

因为simpleURL函数最终会直接被调用，其实现中并没有依赖this的指向，所以在使用bind的时候，第一个参数传入的是null。紧接着传入了simpleURL函数的前两个参数，对它进行柯里化，得到了只接受path作为参数的一个新的函数，并传入到map方法中作为回调函数。
 
那么从上面的例子中，也可以看出在什么场景下适合对函数进行柯里化。当需要多次调用的函数接受的参数过多且大多数都是固定的情况下，可以考虑使用bind对它进行柯里化。
 
## 总结

1. 当场景合适的情况下，考虑使用bind实现函数/方法的柯里化
2. 当function以函数的形式调用时(即不考虑this的指向)，可以使用null或者undefined作为bind的第一个参数

