---
title: '[Java Performance] 字符串拼接注意事项'
date: 2014-09-26 12:32:00
categories: [编程语言, Java]
tags: [Java, Performance, 字符串]
---

## 字符串拼接(String Concatenation)

```java
// 编译器优化前  
String answer = integerPart + "." + mantissa;  
  
// 编译器优化后  
String answer = new StringBuilder(integerPart).append(".").append(mantissa).toString();  
```

因为编译器会对字符串的拼接操作进行优化，所以在同一条语句中使用字符串拼接操作对性能并没有负面影响。正因为编译器在幕后的优化，在任何场景下都使用StringBuilder替代“+”操作符来进行字符串拼接是没有必要的。

<!-- More -->

但是，当字符串的拼接是由多条语句(或者循环)完成的，就有问题了：

```java
// 编译器优化前  
String answer = integerPart;  
answer += ".";  
answer += mantissa;  
  
// 编译器优化后  
String answer = new StringBuilder(integerPart).toString();  
answer = new StringBuilder(answer).append(".").toString();  
answer = new StringBuilder(answer).append(mantissa).toString();  
```

被编译器优化的代码中，中间的String对象以及StringBuilder对象实际上都是不需要的。此时，可以考虑将以上的拼接语句合并成一条拼接语句。或者直接显式地使用StringBuilder对象。

## 总结

1. 当字符串拼接使用一条语句完成时，性能和使用StringBuilder时相当。
2. 对于使用多条语句完成字符串拼接时，考虑合并这些语句或者显式使用StringBuilder。
