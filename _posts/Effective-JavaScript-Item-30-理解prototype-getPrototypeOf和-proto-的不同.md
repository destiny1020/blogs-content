---
title: '[Effective JavaScript] Item 30 理解prototype, getPrototypeOf和__proto__的不同'
date: 2014-09-28 11:13:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。
 
## 重点 
 
prototype，getPropertyOf和\_\_proto\_\_是三个用来访问prototype的方法。它们的命名方式很类似因此很容易带来困惑。

它们的使用方式如下：
 
- **prototype**:

	一般用来为一个类型建立它的原型继承对象。比如C.prototype = xxx，这样就会让使用new C()得到的对象的原型对象为xxx。当然使用obj.prototype也能够得到obj的原型对象。
 
- **getPropertyOf**:

	Object.getPropertyOf(obj)是ES5中用来得到obj对象的原型对象的标准方法。
 
- **\_\_proto\_\_**:

	obj.\_\_proto\_\_是一个非标准的用来得到obj对象的原型对象的方法。
	
<!-- More -->
	
为了充分了解获取原型的各种方式，以下是一个例子：

```js
function User(name, passwordHash) {  
    this.name = name;  
    this.passwordHash = passwordHash;  
}  
User.prototype.toString = function() {  
    return "[User " + this.name + "]";  
};  
User.prototype.checkPassword = function(password) {  
    return hash(password) === this.passwordHash;  
};  
var u = new User("sfalken", "0ef33ae791068ec64b502d6cb0191387"); 
```

User函数拥有一个默认的prototype属性，该属性的值是一个空对象。在以上的例子中，向prototype对象添加了两个方法，分别是toString和checkPassword。当调用User构造函数得到一个新的对象u时，它的原型对象会被自动赋值到User.prototype对象。即u.prototype === User.prototype会返回true。
 
User函数，User.prototype，对象u之间的关系可以表示如下：

![](http://img.blog.csdn.net/20140928111134227?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZG1fdmluY2VudA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

上图中的箭头表示的是继承关系。当访问u对象的某些属性时，会首先尝试读取u对象上的属性，如果u对象上并没有这个属性，就会查找其原型对象。
比如当调用u.checkPassword()时，因为checkPassword定义在其原型对象上，所以在u对象上不会找到该属性，查找顺序是u-> u.prototype。
 
前面提到过，getPrototypeOf方法是ES5中用来得到某个对象的原型对象的标准方法。因此：

```js
Object.getPrototypeOf(u) === User.prototype; // true 
```

在一些环境中，同时提供了一个非标准的__proto__属性用来得到某个对象的原型对象。当环境不提供ES5的标准方法getPrototypeOf方法时，可以暂时使用该属性作为替代。可以使用下面的代码测试环境中是否支持\_\_proto\_\_：

```js
u.__proto__ === User.prototype; // true  
```

所以在JavaScript中，类的概念是由构造函数和其原型对象共同完成的。构造函数中负责构造每个对象特有的属性，比如上述例子中的name和password属性。而其原型对象中负责存放所有对象共有的属性，比如上述例子中的checkPassword和toString方法。就像下面这张图表示的那样：

![](http://img.blog.csdn.net/20140928111100937?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZG1fdmluY2VudA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

## 总结

1. 使用C.prototype来决定new C()得到的对象的原型对象。
2. Object.getPrototypeOf(obj)方法是ES5中提供的用于得到某个对象的原型对象的标准方法。
3. obj.\_\_proto\_\_是获取某个对象的原型对象的非标准方法。
4. 在JavaScript中，类的概念是由构造函数和其原型对象共同定义的。

