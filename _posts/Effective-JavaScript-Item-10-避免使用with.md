---
title: '[Effective JavaScript] Item 10 避免使用with'
date: 2014-09-04 17:06:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。

## 重点

- 设计with关键字本来是为了让代码变简洁，但是却起到了相反的效果，比如：

```js
function f(x, y) {  
	with (Math) {  
    return min(round(x), sqrt(y)); // ambiguous references  
	}  
}  
```

<!-- More -->
	
以上的代码中，调用的min，round以及sqrt都是Math上的方法。

如果Math对象上没有以上指定的方法，那么会在with以外的范围去寻找该方法。
 
那么如果Math对象上有两个字段分别是x和y：
	
```js
Math.x = 0;  
Math.y = 0;  
f(2, 9); // 0  
```
	
可以发现，最后的调用结果是0。
	
- 编译器也无法对代码做出优化，会降低对property的查找速度
- 一个with的替代方案是给变量尽可能短的名字，然后还是使用常规地方式进行操作，比如：

```js
function f(x, y) {  
    var min = Math.min, round = Math.round, sqrt = Math.sqrt;  
    return min(round(x), sqrt(y));  
}  
```
	
## 总结

1. 避免使用with关键字
2. 对于频繁使用的对象，使用尽可能简短的变量名
3. 显式地绑定object上的property到local variable，而不是使用with




