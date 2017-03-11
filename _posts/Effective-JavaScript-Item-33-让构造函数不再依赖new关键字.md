---
title: '[Effective JavaScript] Item 33 让构造函数不再依赖new关键字'
date: 2014-10-05 19:13:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为EffectiveJavaScript的读书笔记。

## 重点
 
在将function当做构造函数使用时，需要确保该函数是通过new关键字进行调用的。

```js
function User(name, passwordHash) {  
    this.name = name;  
    this.passwordHash = passwordHash;  
}  
```

<!-- More -->

如果在调用上述构造函数时，忘记了使用new关键字，那么：

```js
var u = User("baravelli", "d8b74df393528d51cd19980ae0aa028e");  
u; // undefined  
this.name; // "baravelli"  
this.passwordHash; // "d8b74df393528d51cd19980ae0aa028e" 
```

可以发现得到的u是undefined，而this.name以及this.passwordHash则被赋了值。但是这里的this指向的则是全局对象。
 
如果将构造函数声明为依赖于strict模式：

```js
function User(name, passwordHash) {  
    "use strict";  
    this.name = name;  
    this.passwordHash = passwordHash;  
}  
var u = User("baravelli", "d8b74df393528d51cd19980ae0aa028e");  
// error: this is undefined  
```

那么在忘记使用new关键字的时候，在调用this.name= name的时候会抛出TypeError错误。这是因为在strict模式下，this的默认指向会被设置为undefined而不是全局对象。
 
那么，是否有种方法能够保证在调用一个函数时，无论使用了new关键字与否，该函数都能够被当做构造函数呢？下面的代码是一种实现方式，使用了instanceof操作：

```js
function User(name, passwordHash) {  
    if (!(this instanceof User)) {  
        return new User(name, passwordHash);  
    }  
    this.name = name;  
    this.passwordHash = passwordHash;  
}  
  
var x = User("baravelli", "d8b74df393528d51cd19980ae0aa028e");  
var y = new User("baravelli", "d8b74df393528d51cd19980ae0aa028e");  
x instanceof User; // true  
y instanceof User; // true  
```

以上的if代码块就是用来处理没有使用new进行调用的情况的。当没有使用new时，this的指向并不是一个User的实例，而在使用了new关键字时，this的指向是一个User类型的实例。
 
另一个更加适合在ES5环境中使用的实现方式如下：

```js
function User(name, passwordHash) {  
    var self = this instanceof User ? this : Object.create(User.prototype);  
    self.name = name;  
    self.passwordHash = passwordHash;  
    return self;  
}  
```

Object.create方法是ES5提供的方法，它能够接受一个对象作为新创建对象的prototype。那么在非ES5环境中，就需要首先实现一个Object.create方法：

```js
if (typeof Object.create === "undefined") {  
    Object.create = function(prototype) {  
        function C() { }  
        C.prototype = prototype;  
        return new C();  
    };  
}  
```

实际上，Object.create方法还有接受第二个参数的版本，第二个参数表示的是在新创建对象上赋予的一系列属性。
 
当上述的函数确实使用了new进行调用时，也能够正确地得到返回的新建对象。这得益于构造器覆盖模式(Constructor Override Pattern)。该模式的含义是：使用了new关键字的表达式的返回值能够被一个显式的return覆盖。正如以上代码中使用了returnself来显式定义了返回值。
 
当然，以上的工作在某些情况下也不是必要的。但是，当一个函数是需要被当做构造函数进行调用时，必须对它进行说明，使用文档是一种方式，将函数的命名使用首字母大写的方式也是一种方式(基于JavaScript语言的一些约定俗成)。

## 总结

1. 使用Object.create来保证一个函数确实是被当做一个构造函数进行调用的，无论new关键字是否被使用了。
2. 对于作为构造函数的函数，在文档中指明这一点来保证其他用户能够正确的使用它。
