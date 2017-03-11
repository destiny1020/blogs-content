---
title: '[Effective JavaScript] Item 11 掌握闭包'
date: 2014-09-05 12:49:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。
 
## 重点

掌握闭包，需要知道以下几个关键点：

- JavaScript允许在当前的function中访问该function外部的变量。

```js
function makeSandwich() {  
	var magicIngredient = "peanut butter";  
	function make(filling) {  
   	return magicIngredient + " and " + filling;  
   }  
	return make("jelly");  
}  
makeSandwich(); // "peanut butter and jelly" 
```
	
以上的make function访问了外部的magicIngredient变量。
	
<!-- More -->
	
- 一个function能够访问该function外部的变量，即使该外部的function已经返回了。听起来有些不可思议，但是别忘了在JavaScript中，function是first-class object（参见Item 19)。

```js
function sandwichMaker() {  
	var magicIngredient = "peanut butter";  
    function make(filling) {  
        return magicIngredient + " and " + filling;  
    }  
    return make;  
}  
var f = sandwichMaker();  
f("jelly"); // "peanut butter and jelly"  
```
	
以上的代码是如何工作的：
	
实际上，JavaScript中的function不仅仅只记录了需要执行的代码的信息，还保存了所有它引用到的变量的信息，这实际上就是闭包的概念。
 
那么以上的makefunction就是一个闭包，它保存了magicIngredient以及filling这两个变量的信息。
 
因此，可以利用这一点来声明更加general-purpose的function：
	
```js
function sandwichMaker(magicIngredient) {  
    function make(filling) {  
        return magicIngredient + " and " + filling;  
    }  
    return make;  
}  
var hamAnd = sandwichMaker("ham");  
hamAnd("cheese"); // "ham and cheese"  
hamAnd("mustard"); // "ham and mustard"  
```
	
闭包是JavaScript中最显著和优雅的特性，JavaScript甚至提供了一个更简洁的方式来创建闭包，叫做Function Expression：
	
```js
function sandwichMaker(magicIngredient) {  
    return function(filling) {  
        return magicIngredient + " and " + filling;  
    };  
}  
```
	
注意到以上的functionexpression是匿名的，因为这里需要的只是返回一个function(也就是闭包)，用来引用一些信息。当然，function expression是可以有名字的，在Item 14中会进行介绍。

- 闭包能够更新外部值。实际上闭包只是保存了外部值的一个引用，而不是拷贝了它们的值。

```js
function box() {  
    var val = undefined;  
    return {  
        set: function(newVal) { val = newVal; },  
        get: function() { return val; },  
        type: function() { return typeof val; }  
    };  
}  
var b = box();  
b.type(); // "undefined"  
b.set(98.6);  
b.get(); // 98.6  
b.type(); // "number" 
```

以上的box函数返回了一个对象，其中含有三个闭包，每个闭包都引用到了其外部的val变量，其中set闭包也能够对val变量进行修改。
 
## 总结

1. 函数能够访问到其外部的变量。
2. 闭包在创建它们的函数返回之后，还能够被访问到。（Closures can outlive the function that creates them.）
3. 闭包内部会存储所使用的外部变量的引用，并且能够获取和修改它们的值。