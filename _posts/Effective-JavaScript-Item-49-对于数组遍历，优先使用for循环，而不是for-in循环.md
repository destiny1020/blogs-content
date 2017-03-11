---
title: '[Effective JavaScript] Item 49 对于数组遍历，优先使用for循环，而不是for..in循环'
date: 2014-11-11 09:49:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。
 
## 重点 

对于下面这段代码，能看出最后的平均数是多少吗？

```js
var scores = [98, 74, 85, 77, 93, 100, 89];  
var total = 0;  
for (var score in scores) {  
    total += score;  
}  
var mean = total / scores.length;  
mean; // ? 
```

通过计算，最后的结果应该是88。

<!-- More -->
 
但是不要忘了在for..in循环中，被遍历的永远是key，而不是value，对于数组同样如此。因此上述for..in循环中的score并不是期望的98， 74等一系列值，而是0， 1等一系列索引。
 
所以你也许会认为最后的结果是：
(0 + 1+ …+ 6) / 7 = 21
 
但是这个答案也是错的。另外一个关键点在于，for..in循环中key的类型永远都是字符串类型，因此这里的+操作符执行的实际上是字符串的拼接操作：
 
最后得到的total实际上是字符串00123456。这个字符串转换成数值类型后的值是123456，然后再将它除以元素的个数7，就得到了最后的结果：17636.571428571428
 
所以，对于数组遍历，还是使用标准的for循环最好：

```js
var scores = [98, 74, 85, 77, 93, 100, 89];  
var total = 0;  
for (var i = 0, n = scores.length; i < n; i++) {  
    total += scores[i];  
}  
var mean = total / scores.length;  
mean; // 88  
```

毕竟对于这种for循环，开发人员是再熟悉不过了。绝对不会将索引变量i当做是值。并且标准的for循环也能够保证循环的顺序，保证这一点对于浮点数的算术操作十分重要。
 
在以上的代码中，也运用到了一个小优化。就是在循环开始前计算好了数组的长度作为循环边界。当在for循环中不会对数组本身进行添加/删除元素操作时，能够稍微提升一点性能，这样就不会在每次循环都对集合的长度进行查询了。
 
## 总结

1. 当遍历数组时，使用标准的for循环，而不要使用for..in循环。
2. 在必要的场合考虑预先保存数组的长度，以提高性能。
