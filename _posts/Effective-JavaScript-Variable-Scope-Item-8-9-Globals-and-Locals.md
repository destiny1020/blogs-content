---
title: '[Effective JavaScript] Variable Scope Item 8-9 Globals and Locals'
date: 2014-08-22 12:47:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。

## Item 8：少用全局对象
 
### 重点

1. 全局对象能够带来便利，但是有经验的程序员都会视图避免它。因为它会带来潜在的命名冲突的风险
2. 全局变量是维系不同模块之间的纽带，模块之间只能通过全局变量来访问对方提供的功能
3. 能使用局部变量的时候，绝不要使用全局变量
4. 在browser中，this关键字会指向全局的window对象
5. 两种用来改变全局对象的方式，通过var关键字声明以及给全局对象设置属性(通过this关键字)
6. 通过全局对象进行针对当前运行环境的特性检测(Feature Detection)，比如在ES5中提供了一个JSON对象用来操作JSON数据，那么可以通过if(this.JSON)来判断当前运行环境是否支持JSON

<!-- More -->
 
### 总结

1. 避免声明全局变量
2. 尽量使用局部变量
3. 避免向全局对象中添加属性
4. 利用全局变量来进行特性检测
 
---
 
## Item 9：总是声明局部变量
 
### 重点

1.隐式声明的全局变量比全局变量更加麻烦。比如：

```js
function swap(a, i, j) {  
    temp= a[i]; // global  
    a[i]= a[j];  
    a[j]= temp;  
}  
```
	
上述的temp就会被隐式地声明成一个全局变量。

2.使用lint工具来检测JavaScript代码中是否有隐式声明的全局变量
 
### 总结

1. 总是记得通过var关键字来声明局部变量
2. 使用lint工具来确保没有隐式声明的全局变量
