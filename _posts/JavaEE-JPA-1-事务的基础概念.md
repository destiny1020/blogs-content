---
title: '[JavaEE - JPA] 1. 事务的基础概念'
date: 2016-09-17 23:34:00
categories: ['工具, 库与框架', JPA]
tags: [JavaEE, JPA, 事务]
---

现在任何应用都需要数据持久化。否则就不算是一个完整的应用。那么对于一个数据持久化而言，最重要的无外乎两方面：

1. 事务管理(Transaction Management)
2. 对象关系映射(Object Relational Mapping)

本文作为JPA(Java Persistence API)这一系列文章的首篇，就来先谈谈事物管理相关的一些概念和基础。

<!-- More -->

## 事务(Transaction, TX)

事务管理，事务管理，管理的是事务。那么事务又究竟是个什么呢。

比较标准的定义可以参考[英文Wiki](https://en.wikipedia.org/wiki/Transaction)以及[百度百科](http://baike.baidu.com/link?url=ppfRV5K2ihgTHCpE127i83eTkl9qt2j_WaZEOtH9OyBKGXTWa_99WTq6_qtyQ5wh4xtuy0gk7AADTCkQM6GPSXwHzyCvmw7KzvpCzTgYd0O)。

这里尝试用比较好理解的方式来解释一下什么是事务。

我们都知道不管多复杂的代码逻辑最终都是由一行行的代码所组成的。这些代码的执行顺序有先有后，在一个执行单元(如今一般是线程)内部，绝对不可能出现同时执行两个操作的情况，那么当前一个操作出现错误的时候(比如抛出了异常)，往往后面的操作就无法执行下去了。如果这两个操作在逻辑上是一个整体，比如我们都知道的银行转帐问题，那么问题就来了。银行转帐粗略可以分为下面两个行为(不考虑查询过程)：

1. 发起账户的金额减去转账金额
2. 目标账户的金额加上转账金额

一个成功的转账操作，上面的两个行为必然都要成功。不会允许行为1成功，而行为2失败的情况存在。那么如何保证这一点呢？答案就是通过事务。

所谓的事务，实际上是一种抽象。它将一系列的细微操作组合起来形成一个整体上的操作，这个整体上的操作要么成功，要么失败，不存在除此之外的任何其它状态。因为处于其它状态就好比上述银行转账例子中的行为1成功，行为2失败这种状态，是万万不可在现实的金融系统中出现的，否则世界岂不乱了套？

所以从上面的例子中，我们可以发现事务最重要的一个特点 - 原子性(Atomicity)。这个整体操作就是一个原子操作，要么其中所有的操作所有都成功，要么所有的都失败。

而除了原子性之外，一般的事务还有另外3个特点：

- 一致性(Consistency)：事务结束后，参与其中的数据应该处于一种能够解释的通的状态。不应该处于一种"错乱"的状态。比如转账前后，参与到过程中的两个账户的总金额应该保持相等(不考虑万恶的手续费)。

- 隔离性(Isolation)：在一个事务正在进行的过程中，对于变更只有在该事务内部才可见。在事务成功提交之前，事务外部对于这个变更是不可见的。比如说，现在银行系统有一个查询转账次数的统计字段，在转账事务的过程中，肯定需要对这个字段进行+1的操作。那么这个操作在转账事务未提交之前，银行的统计程序是没办法得到变更后的最新数据的。只有当转账确确实实成功提交之后，这个最新的数据才生效，才对外部可见。

- 持久性(Durability)：在事务内执行的变更操作在事务成功提交后仍然生效。(这个很好理解，要不岂不是白干了)

将上面的4个特点的首字母组合起来，就是大名鼎鼎的ACID。这个ACID就是用来描述事务的特点的。虽然我们在后面的讨论中会发现尽管JavaEE应用中的事务通常都是满足ACID几个特点的。但是随着技术的发展和更迭，并不是所有的事务都能够满足ACID。尤其是现在互联网和大数据的时代背景下，由于数据量激增，对于某些应用场景已经没有办法让事务满足所有的4个特点。往往都会根据场景的特点来牺牲掉某个特点，来换取更佳的性能让用户能够满意。

## JavaEE中的事务

既然本文是作为介绍和讨论JPA的首篇文章，那么就必然需要提及JavaEE环境下的事务。毕竟JPA也只是JavaEE整体生态环境下的一个用于描述数据持久化的规范而已。

JavaEE中的事务可以分为两种类型：

1. Resource-local事务
2. Container事务

 Resource-local事务，翻译成中文就是"本地资源"事务。这是最基本的事务类型，直接和JDBC的DataSource接口打交道，因此本质上而言它就是数据库事务。所以即使不在JavaEE这个环境下，比如JavaSE中也是能够使用这种事务类型的。

而Container事务就不同了，根据名字就可以知道它是依赖于容器(Container)的，对于实现了JavaEE标准的应用服务器而言，Container事务一般指的就是使用了JTA(Java Transaction API)的事务。这种事务的特点是能够将一系列的企业级资源涵盖到一个事务中，比如我们常见的数据库，消息队列等等。毕竟企业级应用比较复杂，数据可能分散到很多个数据源中，保存和修改这些数据的时候要保证它们的ACID性质还是需要一定代价的。

### 事务划分(Transaction Demarcation)

我们已经知道了事务实际上是将一系列操作打了个包，形成了一个"原子"操作。那么事务划分所要解决的问题就是如何规定事务从哪里开始，到哪里结束。对于企业级应用的开发而言，事务划分可以算是比较重要的一环，如果事务划分的不恰当则很容易引起数据错乱以及性能下降。

接下来我们来看看上面谈到的两种事务类型是如何划分的。

#### Resource-local事务

对于这种最基本的事务类型，都是由开发人员直接通过编码地方式来完成事务划分。比如下面这类常见操作：

```java
tx.begin();    // 事务开始
// ......
tx.commit();  // 事务提交
```

所以事务划分听上去有点吓人，实际上放到Resource-local事务类型中来，就是两行代码而已(如果考虑到可能出现的tx.rollback()，则是三行代码)。

#### Container事务

对于容器中的事务，除了像上面那样由开发人员直接划分，还多了一种选择，就是让容器来帮你划分。归纳一下就是下面的两种方案：

1. 使用JTA接口在应用中编码完成显式划分
2. 在容器的帮助下完成自动划分

由于JPA作为JavaEE规范的一部分，对同属于JavaEE规范中的EJB作了充分考虑，因此对于EJB而言，上面的两种方案可以被具体描述为：

对于第一种情况，利用JTA提供的接口完成的划分，可以被称为基于Bean的事务(Bean-managed Transaction，BMT)。
对于第二种情况，利用容器完成事务的自动划分的，可以被成为基于容器的事务(Container-managed Transaction，CMT)。

而我们都知道EJB在2.x时代由于其自身十分笨重，开发效率比较低而被疯狂吐槽。所以才出现了时至今日都十分流行的Spring Framework。好不容易到了EJB 3.x时代，虽然它提高了不少，但是由于口碑比较差加上Spring Framework的出色表现，导致真正使用EJB来进行企业级Java开发的人数还是比较少。

那么将上面两种方案放到Spring Framework这个语境中，是这样描述的：

对于第一种情况，被称为编程式的事务管理(Programmatic Transaction Management)。
对于第二种情况，被称为声明式的事务管理(Declarative Transaction Management)。

所以不要拘泥于具体的称呼，还是要还原到问题的本质。而规范就是定义这个本质的。

## 总结

本篇文章首先介绍了事务是什么，然后提到了非常著名的ACID性质。
紧接介绍了JavaEE中的事务类型以及事务划分的概念。
在下一篇文章中将继续介绍事务划分在EJB以及Spring中的具体实现方式。



