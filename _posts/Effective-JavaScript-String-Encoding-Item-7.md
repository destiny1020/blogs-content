---
title: '[Effective JavaScript] String Encoding Item 7'
date: 2014-08-21 12:04:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。
 
提起Unicode，也许许多程序员都会觉得这玩意很麻烦，可是本质上，Unicode并不复杂。世界上每种语言的每一个文字都被一个整形数值表示，范围是0到1114111，这个值在Unicode术语中被称为Code Point。在字符到整形数值的映射上，Unicode和其它编码方式诸如ASCII并没有区别。

<!-- More -->

但是，Unicode存在多种编码方式，而ASCII只有一种方式：

|  字符集 | 编码方式  |
|---|---|
|  ASCII |  ASCII Encoding, e.g. A -> 65 |
|  Unicode |  UTF-8, UTF-16, UTF-32, etc |

那么为什么Unicode有这么多种编码方式呢？因为在不同情况下对操作的时间和空间要求是不一样的。
 
而在设计之初，Unicode估计所有的Code Points能够被2的16次方，也就是65536来表示。这种编码方式就是UCS-2，它是最初的对于Unicode的16位编码方式。通过这种方式，每一个Code Point都可以用一个16位的值进行表示，该表示被称为Code Unit。这种表示方式的优点在于，对Unicode字符串的索引操作都可以在常数时间内完成，因为所有的字符都是由16位，也就是2个比特表示。
 
因为这种编码方式的便利性，所以一些平台诸如Java，JavaScript都采用了它。因此，JavaScript的字符串每一个字符都是由2个比特表示的。

而随着Unicode字符集的扩展，65536已经满足不了需求了，目前Unicode字符集中字符的数量已经超过了2的20次方。因此，新增加的部分被组织到由17个2的16次方所组成的子范围内。（17 * 2^16 = 1114112，所以目前Unicode的Code Point范围是0-1114111）
 
第一个子范围，用来容纳原来UCS-2中的字符集，它也被称为Basic Multilingual Plane(BMP)。剩下的16个子范围，被称为Supplementary Planes。
 
为了表示更多的字符，UCS-2的继任者UTF-16，是这样设计的：
对于Code Point大于等于65536的字符，由一对16位的Code Unit表示。对于Code Point小于65536的字符，还是只需要1个16位的Code Unit表示。因此，UTF-16是一种变长编码方式，所以在对Code Point做indexing操作也不是常数时间了。它通常都需要从字符串的开始往后搜索。
 
对于JavaScript，字符串的length属性，charAt以及charCodeAt方法，都是在Code Unit的基础上工作，而不是Code Point。因此，当JavaScript需要表示出于Supplementary Plane中的Code Point时，它都会使用两个CodeUnit来表示，简而言之：
JavaScript字符串是由16位的Code Unit组成的。
 
所以，当需要处理BMP以外的Code Point时，会带来一些问题，因为你不能依赖于length属性，charAt以及charCodeAt方法了。这时候可以考虑使用一些成熟的第三方库。
 
## 总结

1. JavaScript的字符串由16比特的Code Unit组成，而不是由Unicode Code Point组成。
2. 大于等于65536的Code Point在JavaScript中由两个Code Units组成，被称为Surrogate Pair。
3. Surrogate Pair会影响到length，charAt，charCodeAt以及正则表达式中 . 的工作方式。
4. 处理Code Point超过65535的字符串时，考虑使用第三方的库并查阅它的文档。
