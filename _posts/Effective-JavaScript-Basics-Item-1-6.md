---
title: '[Effective JavaScript] Basics Item 1-6'
date: 2014-08-19 19:56:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

## Item 1: 了解你正在使用的JavaScript

### 重点

1. 如果使用了strict mode，那么需要将你的代码在ES5环境中进行测试
2. use strict只在script或者function的最开始处才能被识别，所以在进行script拼接的时候需要注意
3. 永远不要将non strict和strict的scripts进行拼接
4. 如果确实需要将non strict和strict的scripts进行拼接，可以考虑使用Immediately Invoked Function Expression(IIFE)

### 总结

1. 确定你的应用需要支持什么版本的JavaScript
2. 确保所有环境都能支持你使用的JavaScript特性
3. 确保在检查strict模式的环境中测试你的代码
4. 在拼接scripts的时候注意它们是non strict还是strict模式

<!-- More -->

## Item2：理解JavaScript的浮点数

### 重点
1. JavaScript中的所有数值都是遵循IEEE754规范的双精度浮点数
2. 所有位操作的工作方式都是首先将浮点数转换成整形数，然后执行操作，最后再将结果转换成浮点数
3. 浮点数运算的结果也许不能满足交换律，比如：
	
	(0.1 + 0.2) + 0.3 的结果和 0.1 + (0.2 + 0.3)的结果分别是：
0.600000000000001 和 0.6

4. 在精度重要的情况下，使用最小的度量单位作为基准，比如表示1元钱时，可以使用100这个值来表示，因为最小的度量单位是分

### 总结

1. JavaScript中的数值是双精度浮点数
2. 整型数值在JavaScript中只是浮点数的一个子集，而不是一个单独的数据类型
3. 位操作将数值当做32位的有符号整形数
4. 注意浮点运算的精度问题

## Item 3：注意隐式的强制转换

### 重点

1. 算术运算符-，*，/，%以及位操作符都会首先尝试将参数转换为数值然后再进行计算
2. 运算符+比较特殊，因为它还重载了字符串拼接的功能
3. 注意null和undefined的在转换到数值的结果，分别是0和NaN
4. NaN的特殊性，它不等于它自己，同时isNaN也有其问题，它首先会尝试将参数转换为数值类型，所以IsNaN("foo")的结果也是true。更好的检测方式是return a !== a
5. object通过调用toString方法来得到string类型，通过调用valueOf方法来得到number类型
6. 在面对+操作符时，因为+既能够操作number类型，也能够操作string类型，所以在使用+操作两个objects的时候，会面临两难。但是JavaScript会选择使用valueOf方法，而不是toString方法
7. 如果你的对象并不是数值类型，不要实现valueOf方法
8. 7个falsy值：false，0，-0，“”，NaN，null以及undefined，所有其他值都是truthy

### 总结

1. 类型错误可能会因为隐式强制转换而被忽略掉
2. 根据参数类型，+会执行加法操作或者字符串连接操作
3. object通过valueOf方法和toString方法来强转成number和string类型
4. 拥有valueOf方法的object应该实现一个toString方法来提供该数值的一个字符串表达
5. 使用typeof或者直接和undefined进行对于undefined值得判断，而不是依赖于truthy判断，因为0或者空字符串在truthy判断时返回的和undefined返回的相同，都是false

## Item 4：优先使用Primitives而不是Object Wrapper

### 重点

1. JavaScript中只有5种primitives：
	
	boolean，number，string，null以及undefined 
	
	但是需要注意的是：typeof null 返回的结果却是object
2. 在必要的时候，primitive类型会被自动转换成为object类型，比如调用"hello".toUpperCase()，实际上hello字符串首先会被创建一个string对象
3. 不要给primitive设置任何属性，尽管这种行为不会报错，但设置的属性并不会生效，因为这个属性是赋给了某个中间对象，并没有办法拿到这个中间对象的引用
 
### 总结

1. 在比较是否相等的时候，primitive和object wrapper的行为并不一致，这一点类似其它的编程语言如Java
2. 设置或者读取primitive的属性的时候，会创建值为该primitive的一个object wrapper

## Item 5：避免在比较不同类型时使用==

### 重点

1. 尽量不要依赖参与到比较运算中参数会被自动转换为number类型这一特性，显式使用Number进行构造或者使用单元+操作符更加合适，比如：

	`+form.month.value === today.getMonth() + 1`
	
2. 优先使用strict equality操作符，即===，这样能够告诉读者在这个比较运算中不涉及到隐式转换
3. 注意一些非常微妙的自动转换规则：
	1. 对于非Date类型的object，会先尝试通过valueOf进行转换，如果没有valueOf则通过toString
	2. 对于Date类型的object，会先尝试通过toString进行转换，如果没有toString则通过valueOf

### 总结

1. ==这一非严格的比较操作符会使用一套令人费解的隐式转换规则来进行比较运算
2. 优先使用===，这样能够告诉读者这个比较操作不依赖和涉及恼人的隐式转换
3. 对于不同类型的比较，使用自定义的转换以及比较逻辑，而不是依赖于隐式转换

## Item 6：了解分号插入的限制

### 重点

尽量不要依赖于分号的自动插入。在需要插入分号的时候就手工插入。编码习惯好的话，很少会遇到这个问题。作为了解即可。
 
### 总结

1. 分号的inferred只会在以下几种情况中生效：
	1. 在 } 之前
	2. 行末
	3. 程序的末尾
2. 只有当下一个token无法被解析的时候才会被inferred
3. 在后续语句以 ( [ + - / 开头的时候，不要在前面省略分号
4. 在对scripts进行拼接的时候，在scripts间显式插入分号
5. 注意restricted productions，即在return，throw以及break以及可选的参数间（比如return可以接上一个可选的参数作为返回值）不要插入换行
6. 在for循环中，不要依赖分号的自动inferred

