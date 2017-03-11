---
title: '[Effective JavaScript] Item 18 理解Function, Method, Constructor调用之间的区别'
date: 2014-09-11 14:18:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

系列作为Effective JavaScript的读书笔记。
 
## 重点 
 
Function绝对是JavaScript中的重中之重。在JavaScript中，Function承担了procedures, methods, constructors甚至是classes以及modules的功能。
 
在面向对象程序设计中，functions，methods以及class constructor往往是三件不同的事情，由不同的语法来实现。但是在JavaScript中，这三个概念都由function来实现，通过三种不同的模式。
 
最简单的使用模式就是function call：

```js
function hello(username) {  
    return "hello, " + username;  
}  
hello("Keyser Söze"); // "hello, Keyser Söze"  
```

<!-- More -->

而methods这一概念在JavaScript中的表现就是，一个对象的属性是一个function：

```js
var obj = {  
    hello: function() {  
        return "hello, " + this.username;  
    },  
    username: "Hans Gruber"  
};  
obj.hello(); // "hello, Hans Gruber"  
```

以上代码中，比较关键的部分是：使用了this关键字，这里的this指向的是obj对象。你也许会认为正因为hello方法定义在了obj对象上，所以this指向的是obj。但是下面这段代码：

```js
var obj2 = {  
hello: obj.hello,  
    username: "Boo Radley"  
};  
obj2.hello(); // "hello, Boo Radley"  
```

真正的行为是，调用本身才会决定this会绑定到哪个对象，即：
obj1.hello()会将this绑定到obj1，obj2.hello()则会将this绑定到obj2。
 
正因为this绑定的这种规则，在下面的用法也是可行的：

```js
function hello() {  
    return "hello, " + this.username;  
}  
  
var obj1 = {  
    hello: hello,  
    username: "Gordon Gekko"  
};  
obj1.hello(); // "hello, Gordon Gekko"  
  
var obj2 = {  
    hello: hello,  
    username: "Biff Tannen"  
};
obj2.hello(); // "hello, Biff Tannen"  
```

但是，在一个普通的函数中，如上面的hello函数，使用this关键字是不太好的方式，当它被直接调用的时候，this的指向就成了问题。在这种情况下，this往往被指向全局对象(GlobalObject)，在浏览器上一般就是window对象。
而这种行为是不确定和没有意义的。
 
所以在ES5标准中，如果使用了strict mode，那么this会被设置为undefined：

```js
function hello() {  
    "use strict";  
    return "hello, " + this.username;  
}  
hello(); // error: cannot read property "username" of undefined  
```

以上这种做法是为了让潜在的错误更快的暴露出来，避免了误操作和难以找到的bug。
 
function的第三种使用模式就是讲它作为constructor：

```js
function User(name, passwordHash) {  
    this.name = name;  
    this.passwordHash = passwordHash;  
}  
var u = new User("sfalken",  
    "0ef33ae791068ec64b502d6cb0191387");  
u.name; // "sfalken"  
```

使用new关键将function作为constructor进行调用。和function以及method调用不一样的是，constructor会传入一个新的对象并将它绑定到this，然后返回该对象作为constructor的返回值。而constructor function本身的作用就是为了初始化该对象。
 
## 总结

1. Method调用本身会决定this绑定到哪个对象
2. Function调用中，this会指向全局对象，在浏览器中，通常是window对象
3. Constructor调用通过new关键字完成，会将this绑定到一个新的对象上






