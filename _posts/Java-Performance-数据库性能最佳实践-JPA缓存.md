---
title: '[Java Performance] 数据库性能最佳实践 - JPA缓存'
date: 2014-10-22 10:13:00
categories: [编程语言, Java]
tags: [Java, Performance, 数据库, JPA, 最佳实践, 缓存]
---

## JPA缓存(JPA Caching)

JPA有两种类型的缓存：

- EntityManager自身就是一种缓存。事务中从数据库获取的和写入到数据库的数据会被缓存(什么样的数据会被缓存，在后面有介绍)。在一个程序中也许会有很多个不同的EntityManager实例，每一个实例运行着不同的事务，拥有着它们自己的缓存。
- 当EntityManager提交一个事务后，它缓存的所有数据就会被合并到一个全局的缓存中。所有的EntityManager都能够访问这个全局的缓存。

全局缓存被称为二级缓存(Level 2 Cache)，而EntityManager拥有的本地缓存被称为一级缓存(Level 1 Cache)。所有的JPA实现都拥有一级缓存，并且对它没有什么可以调优的。而二级缓存就不同了：大多数JPA实现都提供了二级缓存，但是有些并没有把启用它作为默认选项，比如Hibernate。一旦启用了二级缓存，它的设置会对性能产生较大的影响。

<!-- More -->

只有当使用实体的主键进行访问时，JPA的缓存才会工作。这意味着，下面的两种获取方式会将获取的结果放入到JPA的缓存中：

- 调用find()方法，因为它需要接受实体类的主键作为参数
- 调用实体类型的getter方法来得到关联的实体类型，本质上，获取关联的实体对象也是通过关联对象的主键得到，因为在数据库的表结构中，存放的是该关联对象的外键信息。

那么当EntityManager需要通过主键或者关联关系获取一个实体对象时，它首先会去二级缓存中寻找。如果找到了，那么它就不需要对数据库进行访问了。

通过查询(JPQL)方式得到的实体对象是不会被放到二级缓存中的。然而在一些JPA实现中也会将查询得到的结果放入到缓存中，但是只有当相同的查询再次被执行时，这些缓存才会起作用，所以即使JPA的实现支持查询缓存，查询返回的实体也不会被存储在二级缓存中，因此也就不能被诸如find()等方法利用了。

通过下面的一段代码对二级缓存和查询进行性能测试：

```java
EntityManager em = emf.createEntityManager();
Query q = em.createNamedQuery(queryName);
List<StockPrice> l = q.getResultList(); // SQL Call 1
for (StockPrice sp : l) {
    // ... process sp ...
    if (processOptions) {
        Collection<? extends StockOptionPrice> options = sp.getOptions(); // SQL Call 2
        for (StockOptionPrice sop : options) {
            // ... process sop ...
        }
    }
}
em.close();
```

以上代码通过一个命名查询来得到StockPrice实体对象。 布尔变量processOptions用来控制是否遍历关联的StockOptionPrice实体对象。

### 缓存和懒加载

```java
@NamedQuery(name="findAll", query="SELECT s FROM StockPriceImpl s ORDER BY s.id.symbol")

@OneToMany(mappedBy="stock")
private Collection<StockOptionPrice> optionsPrices;
```

在默认情况下，对于StockPrice关联的StockOptionPrice，由于是一对多的关联方式，后者的加载类型是懒加载，运行

|测试用例	|首次执行|	后续执行|
| --- | --- | --- |
|默认缓存策略 + 懒加载	|61.9s (33,409 SQL调用)|	3.2s (1 SQL 调用)|
|默认缓存策略 + 懒加载 + 不遍历关联对象|	5.6s (1 SQL 调用)|	2.8s (1 SQL 调用)|

当需要遍历关联对象时，在首次执行时产生了大量SQL调用，这是由于对于每个StockPrice实例，都需要遍历其StockOptionPrice集合，因此产生了：128 * 261 = 33408次SQL调用。再加上获取StockPrice的一次命名查询，所以一共是33409次。但是在后续执行时，只会发生一次命名查询导致的SQL调用，这是因为StockOptionPrice此时全部都已经被存储到二级缓存中(由关联关系和find方法得到的实体对象会被保存到二级缓存中，而查询结果则不会被保存)，不需要再对数据库进行访问。

当不需要遍历关联对象时，每次执行都只会产生一次SQL调用。同时注意到对于此测试用例，首次执行仍然比后续执行要慢整整一倍，这是因为编译器的“热身”也会在首次执行期间进行(关于JIT编译器的性质，请查看相关章节)。

### 缓存和立即加载

当StockOptionPrice的加载方式切换成立即加载后，得到的测试数据如下：

|测试用例	|首次执行	|后续执行|
| --- | --- | --- |
|默认缓存策略 + 立即加载|60.2s (33,409 SQL调用)|	3.1s (1 SQL 调用)|
|默认缓存策略 + 立即加载 + 不遍历关联对象|	60.2s (33,409 SQL 调用)|	2.8s (1 SQL 调用)|

此时，无论是否选择遍历关联对象，都会发生33409次SQL调用。因为在执行命名查询得到每个StockPrice对象后，就会顺便调用StockOptionPrice的getter方法来得到关联对象。此时得到的StockOptionPrice对象会被存储到二级缓存中，因此在后续执行中不会再触发SQL调用。

### JOIN FETCH和缓存

如果在命名查询中使用JOIN FETCH：

```java
@NamedQuery(name="findAll", query="SELECT s FROM StockPriceEagerLazyImpl s " + "JOIN FETCH s.optionsPrices ORDER BY s.id.symbol")
```

|测试用例	|首次执行	|后续执行|
| --- | --- | --- |
|默认配置|	61.9s (33,409 SQL调用)|	3.2s (1 SQL 调用)|
|JOIN FETCH	|17.9s (1 SQL 调用)|	11.4s (1 SQL 调用)|
|JOIN FETCH + 查询缓存|	17.9s (1 SQL 调用)|	1.1s (0 SQL 调用)|

当使用了JOIN FETCH后，性能得到了非常大的提升。虽然查询的数据量是相同的，但是发生的SQL调用剧减到了1，这也是性能得以大幅提升的首要原因。但是，由于缺少查询缓存，在后续调用的时候仍然需要较长的时间(同样地，执行时间从17.9s -> 11.4s是因为首次执行期间JIT编译器需要“热身”)。

所以在最后一个测试用例，当开启了查询缓存后，后续执行的时间大幅缩短到1.1s，同时没有发生SQL调用！这是一个使用查询缓存的典型例子，但是需要注意只有当查询使用的参数完全相同时，查询缓存才会起作用。

### 避免查询

根据二级缓存的特点，如果不使用查询，那么得到的所有对象都会被保存到二级缓存中。那么当程序运行一段时间后，随着对象都被缓存，需要执行的SQL语句就越来越少，程序的运行速度也就越来越快了：

```java
EntityManager em = emf.createEntityManager();
ArrayList<String> allSymbols = ... all valid symbols ...;
ArrayList<Date> allDates = ... all valid dates...;
for (String symbol : allSymbols) {
    for (Date date = allDates) {
        StockPrice sp = em.find(StockPriceImpl.class, new StockPricePK(symbol, date);
        // ... process sp ...
        if (processOptions) {
            Collection<? extends StockOptionPrice> options = sp.getOptions();
            // ... process options ...
        }
    }
}
```

测试结果如下所示：

|测试用例	|首次执行	|后续执行|
| --- | --- | --- |
|默认配置|	61.9s (33,409 SQL调用)|	3.2s (1 SQL 调用)|
|无查询	|100.5s (66,816 SQL 调用)	|1.19s (0 SQL 调用)|

首次执行会产生66816次SQL调用，其中33408次是调用find方法时产生的，另外33408次时调用getOptions方法时产生的。在此之后，所有的对象都会被保存到二级缓存中，因此后续执行时，没有SQL被执行。

所以，当使用无查询的策略是，首次执行的时间通常会比较长，这个过程可以被看成是一个“热身”的过程，在“热身”结束之后，程序的性能会提高一个档次。

另外需要注意的一个问题是，即使使用getOptions方法得到的是一个集合对象，这个集合对象的所有元素也会被存储到二级缓存中，不要将它和查询混淆。所以，当希望缓存一个实体对象关联的一组实体对象时，只需要调用相应的getter方法即可，甚至不需要对该集合进行遍历。

### 设置JPA缓存的空间

当JPA缓存占用的内存过多时，它会给GC添加不小的压力。所以JPA缓存的空间需要被仔细设置。但是，JPA规范并没有规定如何设置JPA缓存，所以需要查看对应JPA实现的相关文档。

### 总结

1. JPA的二级缓存会自动地为应用缓存对象。
2. 二级缓存不会保存查询(JPQL)的返回对象，所以当需要缓存对象时，不要使用查询。(或者开启查询缓存)
3. 谨慎使用结合了JOIN FETCH的查询，除非使用的JPA实现支持查询缓存。因为默认情况下，查询会跳过二级缓存。

## JPA只读实体(JPA Read-Only Entities)

尽管JPA规范并没有介绍只读实体，但是在很多JPA实现中，都会这种实体作出相应的优化。 对只读实体的操作在性能上一般都会优于读写实体(Read-Write Entities)。因为对于只读实体，不需要保存它的状态，不需要将它放在事务中，也不需要对它进行加锁。

在Java EE容器中，无论使用的什么JPA实现，只读实体一般都会被支持。应用服务器会保证对这些实体的获取是通过一个特殊的非事务性的JDBC连接来完成。这样做通常都有更好的性能。



