---
title: '[Effective JavaScript] Item 23 永远不要修改arguments对象'
date: 2014-09-19 09:40:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。
 
## 重点 
 
arguments对象只是一个类似数组的对象，但是它并没有数组对象提供的方法，比如shift，push等。因此调用诸如：arguments.shift()，arguments.push()是错误的。
 
在Item 20和Item 21中，知道了函数对象上存在call和apply方法，那么是不是可以利用它们来让arguments也能够利用数组的方法呢：

```js
function callMethod(obj, method) {  
    var shift = [].shift;  
    shift.call(arguments);  
    shift.call(arguments);  
    return obj[method].apply(obj, arguments);  
}  
```

但是以上的方法在下面的应用场景中存在问题：

<!-- More -->

```js
var obj = {  
    add: function(x, y) { return x + y; }  
};  
callMethod(obj, "add", 17, 25);  
// error: cannot read property "apply" of undefined  
```

发生错误的原因是：
arguments对象并不是函数参数的一份拷贝。函数声明的参数和arguments保存的对象存在着引用关系。比如在上面callMethod函数的例子中，声明了两个参数obj和method：

- obj引用的就是arguments[0]
- method引用的就是arguments[1]

而在调用了两次shift.call(arguments)之后，arguments由原来的：

[obj, "add", 17, 25]变成了[17, 25]
 
所以obj的引用从obj本身变成了17，method的引用从"add"变成了25。很显然17[25]得到的结果是undefined，因为根据JavaScript的运算规则，17首先会被转换为Number对象，而这个对象之上并没有25这个属性。
 
上述例子想表达的就是，函数中声明的参数和arguments之间的联系很脆弱，每个声明的参数实际上只是对arguments对象中对应位置的一个引用。

值得注意的是，在ES5的strict mode中，函数声明的参数并不会引用arguments：

```js
function strict(x) {  
    "use strict";  
    arguments[0] = "modified";  
    return x === arguments[0];  
}  
function nonstrict(x) {  
    arguments[0] = "modified";  
    return x === arguments[0];  
}  
strict("unmodified"); // false  
nonstrict("unmodified"); // true  
```

正因为在strict和非strict模式下，函数声明的参数和arguments的关系不一致，所以为了避免出现问题，不去修改arguments对象才是最安全的做法。
 
如果确实需要修改arguments对象，那么可以首先赋值一份arguments对象：

```js
var args = [].slice.call(arguments);  
```

当slice方法不接受任何参数的时候，就会执行复制操作，得到的args也是一个真正的数组对象。同时，args和函数声明的参数之间也没有任何联系了，对它进行操作是安全的。使用这种方式重新实现上面提到过的callMethod函数：

```js
function callMethod(obj, method) {  
    var args = [].slice.call(arguments, 2);  
    return obj[method].apply(obj, args);  
}  
  
var obj = {  
    add: function(x, y) { return x + y; }  
};  
callMethod(obj, "add", 17, 25); // 42  
```

## 总结

1. 永远不要修改arguments对象
2. 使用[].slice.call(arguments)得到arguments对象的一份拷贝，然后对拷贝进行修改