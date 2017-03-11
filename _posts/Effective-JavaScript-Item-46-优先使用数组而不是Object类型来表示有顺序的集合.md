---
title: '[Effective JavaScript] Item 46 优先使用数组而不是Object类型来表示有顺序的集合'
date: 2014-10-28 16:27:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。
 
## 重点 
 
ECMAScript标准并没有规定对JavaScript的Object类型中的属性的存储顺序。
 
但是在使用for..in循环对Object中的属性进行遍历的时候，确实是需要依赖于某种顺序的。正因为ECMAScript没有对这个顺序进行明确地规范，所以每个JavaScript执行引擎都能够根据自身的特点进行实现，那么在不同的执行环境中就不能保证for..in循环的行为一致性了。
 
<!-- More -->
 
比如，以下代码在调用report方法时的结果就是不确定的：

```js
function report(highScores) {  
    var result = "";  
    var i = 1;  
    for (var name in highScores) { // unpredictable order  
        result += i + ". " + name + ": " +  
        highScores[name] + "\n";  
        i++;  
    }  
    return result;  
}  
report([{ name: "Hank", points: 1110100 },  
{ name: "Steve", points: 1064500 },  
{ name: "Billy", points: 1050200 }]);  
// ?  
```

如果你确实需要保证运行的结果是建立在数据的顺序上，优先使用数组类型来表示数据，而不是直接使用Object类型。同时，也尽量避免使用for..in循环，而使用显式的for循环：

```js
function report(highScores) {  
    var result = "";  
    for (var i = 0, n = highScores.length; i < n; i++) {  
        var score = highScores[i];  
        result += (i + 1) + ". " +  
        score.name + ": " + score.points + "\n";  
    }  
    return result;  
}  
report([{ name: "Hank", points: 1110100 },  
{ name: "Steve", points: 1064500 },  
{ name: "Billy", points: 1050200 }]);  
// "1. Hank: 1110100\n2. Steve: 1064500\n3. Billy: 1050200\n" 
```

另一个特别依赖于顺序的行为是浮点数的计算：

```js
var ratings = {  
    "Good Will Hunting": 0.8,  
    "Mystic River": 0.7,  
    "21": 0.6,  
    "Doubt": 0.9  
};  
```

在Item 2中，谈到了浮点数的加法操作甚至不能满足交换律：
(0.1 + 0.2) + 0.3 的结果和 0.1 + (0.2 + 0.3)的结果分别是
0.600000000000001 和 0.6
 
所以对于浮点数的算术操作，更加不能使用任意的顺序了：

```js
var total = 0, count = 0;  
for (var key in ratings) { // unpredictable order  
    total += ratings[key];  
    count++;  
}  
total /= count;  
total; // ?  
```

当for..in的遍历顺序不一样时，最后得到的total结果也就不一样了，以下是两种计算顺序和其对应的结果：

``` 
(0.8 + 0.7 + 0.6 +0.9) / 4 // 0.75
(0.6 + 0.8 + 0.7 +0.9) / 4 // 0.7499999999999999
```
 
当然，对于浮点数的计算这一类问题，有一个解决方案是使用整型数来表示，比如我们将上面的浮点数首先放大10倍变成整型数据，然后计算结束之后再缩小10倍：

```
(8+ 7 + 6 + 9) / 4 / 10    // 0.75
(6+ 8 + 7 + 9) / 4 / 10    // 0.75
```
 
## 总结

1. 在使用for..in循环时，不要依赖于遍历的顺序。
2. 当使用Object类型来保存数据时，需要保证其中的数据是无序的。
3. 当需要表示带有顺序的集合时，使用数组类型而不是Object类型。



