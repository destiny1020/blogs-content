---
title: '[Java 8 Lambda] java.util.stream 简介'
date: 2014-05-15 23:49:00
categories: [编程语言, Java]
tags: [Java, Java 8, Lambda, Stream]
---

包结构如下所示：

![](http://img.blog.csdn.net/20140515234746562?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZG1fdmluY2VudA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

这个包的结构很简单，类型也不多。

<!-- More -->
 
## BaseStream接口

所有Stream接口类型的父接口，它继承自AutoClosable接口，定义了一些所有Stream都具备的行为。
 
因为继承自AutoClosable接口，所以所有的Stream类型都可以用在Java 7中引入的try-with-resource机制中，以达到自动关闭资源的目的。实际上，只有当Stream是通过Socket，Files IO等方式创建的时候，才需要关闭它。对于来自于Collections，Arrays的Stream，是不需要关闭的。
 
## Stream接口

定义了众多Stream应该具有的行为。
最典型的比如filter方法族，map方法族以及reduce方法族，这三个方法是FunctionalProgramming的标志。典型的Map-Filter-Reduce模式便是依靠这三个操作来定义的。
 
与此同时，Stream接口还定义了一些用于创建Stream的static方法，创建的Stream可以是有限的，也可以是无限的。有限的很好理解，而无限Stream是一个新概念，通过generate方法或者iterate方法实现。
 
## IntStream, LongStream 以及 DoubleStream 接口

基于原生类型int, long以及double的Stream。提供了众多类型相关的操作。
典型的例如，sum方法，min/max方法，average方法等。这些方法都是Reduce操作的具体实现。
 
## Collect接口

对于Reduce操作的抽象。此接口中定义了常用的Reduce操作。
其中定义的Reduce操作可以通过串行或者并行的方式进行实现。BaseStream接口中的parallel，sequential，unordered方法提供的高层API使并发程序设计变得非常简洁。
毕竟，Map-Filter-Reduce模式的灵魂就在于并行计算。
 
## Collectors类

提供了众多可以直接使用的Reduce操作。
典型的比如groupingBy以及partitioningBy操作。它们都可以通过串行或者并行的方式进行实现。比如，groupingByConcurrent会使用并行的方式进行grouping操作。
 
## StreamSupport类

提供了底层的一些用于操作Stream的方法，如果不需要创建自己的Stream，一般不需要使用它。
