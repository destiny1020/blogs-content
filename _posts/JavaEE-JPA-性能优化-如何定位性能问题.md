---
title: '[JavaEE - JPA] 性能优化: 如何定位性能问题'
date: 2016-12-03 19:26:00
categories: ['工具, 库与框架', JPA]
tags: [JavaEE, JPA, Performance, 优化]
---

要想解决性能问题，首先得要有办法定位问题，明白问题究竟是什么。

本来JPA的存在目的就是为了让开发人员能够更少地直接操作SQL，但是由于业务自身有其复杂性，如果开发人员不老练，没有踩过许许多多形形色色的坑，是很难写出高质量的JPA代码的，这也是为什么很多人说Hibernate(JPA)入门容易，精通难。实际上不是精通难，而是懒得花那么多精力去研究和发现JPA的潜在性能问题。而JPA的性能问题，可以说99%都是因为JPA Provider(一般使用的都是Hibernate，或者EclipseLink)生成的SQL效率低下或者生成并执行了你意料之外的SQL。

针对这个问题，其实不需要多么复杂的调试工具，一般而言JPA Provider就会提供一些基础的性能分析工具，以Hibernate为例(EclipseLink等其它JPA Provider请参考相关文档)介绍两种最常用的方法。

<!-- More -->

## 打印执行的SQL

这是最直观的方案，要想知道为什么执行效率不符合预期，直接查看执行的SQL不就行了嘛。毕竟很多大神的调试方法也就仅仅是万能的`printf`。(记得是在《编程珠玑》上看到的)
可以在persistence.xml中指定如下：

```xml
<persistence 
    xmlns="http://java.sun.com/xml/ns/persistence"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://java.sun.com/xml/ns/persistence 
    http://java.sun.com/xml/ns/persistence/persistence_1_0.xsd"
    version="1.0">
    <persistence-unit name="<PERSISTENCE UNIT NAME>">
        <properties>
            <property name="hibernate.show_sql" value="true"/>
            <property name="hibernate.format_sql" value="true"/>
            <property name="hibernate.use_sql_comments" value="true"/>
        </properties>
    </persistence-unit>
</persistence>
```

如果你使用的是Spring Boot，那么就更简单了：

```yml
spring:
  jpa:
    properties:
      hibernate.show_sql: true
      hibernate.format_sql: true
      hibernate.use_sql_comments: true
```

注意以上使用了三个属性，分别是`hibernate.show_sql`，`hibernate.format_sql`以及`hibernate.use_sql_comments`。这三个属性是相关联的，其中`hibernate.show_sql`是最基本的，只有当它被设置成`true`的时候设置另外两个才有意义。

一般而言，`hibernate.show_sql`通常都是和`hibernate.format_sql`一起设置成`true`的。这样能够保证打印出来的SQL清晰易读，毕竟打印SQL就已经很费事了，干脆再费事一点把SQL打印地工整一点不是更好吗？反正这两个配置也仅仅是在开发测试以及调优阶段使用的，在生产环境中不要忘记将它们设置成`false`就好了。

那么`hibernate.use_sql_comments`又是干什么用的呢？将它设置成`true`可以打印除了实际生成执行的SQL之外的一些信息。例如下面这段打印信息：

```
Hibernate: 
    /* select
        generatedAlias0 
    from
        DeliveryCompany as generatedAlias0 
    order by
        generatedAlias0.sort asc */ select
            deliveryco0_.name as name1_10_,
            deliveryco0_.sort as sort2_10_ 
        from
            bono_config_delivery_company deliveryco0_ 
        order by
            deliveryco0_.sort asc
```

可以观察到在`select`之前有一段以`/*`和`*/`包围起来的注释信息。这段注释信息实际上就是用于生成SQL的JPQL。这是SQL为select类型的情况，如果是插入数据，显示的日志信息则是这样的结构：

```
Hibernate: 
    /* insert com.xxx.yyy.zzz.SampleClass
        */ insert 
        into
            xxx_sample_class
            (code, icon_url, is_visible, name) 
        values
            (?, ?, ?, ?)
```

注释部分会告诉你是插入的记录完整类型名称。

所以使用`hibernate.use_sql_comments`可以让打印出来的信息更加充分，方便我们的调优工作。

可是这还没完，有时候我们需要在打印信息中查看具体绑定的参数，而不是看到上面那样的`?`占位符。这可以通过配置相应日志级别来实现，因为参数绑定的操作在Hibernate中算是最最基本和频繁的，所以它们的默认日志级别被设置成了`TRACE`，也就是最低的日志级别。一般情况下是不会打印出来的。如果想打印它们，可以配置如下(以logback为例)：

```xml
<logger name="org.hibernate.type" level="TRACE" additivity="false">
	<appender-ref ref="CONSOLE" />
</logger>
```

以上只将日志输出到了控制台上，如果希望打印到其它的媒介上，可以进行相应配置。具体请查看你所使用日志工具的文档。

配置完毕后，如果再次执行上述的插入记录的操作，打印出来的信息会是这样的：

```
Hibernate: 
    /* insert com.xxx.yyy.zzz.SampleClass
        */ insert 
        into
            xxx_sample_class
            (code, icon_url, is_visible, name) 
        values
            (?, ?, ?, ?)
2016-12-03 17:03:05.682 TRACE 39585 --- [io-19999-exec-8] o.h.type.descriptor.sql.BasicBinder      : binding parameter [1] as [VARCHAR] - [T]
2016-12-03 17:03:05.682 TRACE 39585 --- [io-19999-exec-8] o.h.type.descriptor.sql.BasicBinder      : binding parameter [2] as [VARCHAR] - [null]
2016-12-03 17:03:05.683 TRACE 39585 --- [io-19999-exec-8] o.h.type.descriptor.sql.BasicBinder      : binding parameter [3] as [BOOLEAN] - [true]
2016-12-03 17:03:05.684 TRACE 39585 --- [io-19999-exec-8] o.h.type.descriptor.sql.BasicBinder      : binding parameter [4] as [VARCHAR] - [Testing]
```

可以看到每个`?`占位符都对应了一条`TRACE`级别的日志，将实际使用的值给打印了出来。

这个功能在分析一些对特殊数据进行操作时所导致的性能问题是十分有用的。通过查看具体的绑定数据，进行业务逻辑的分析进而得到性能瓶颈所在。

以上就是打印SQL这种方案所对应的一些配置，总结如下：

1. 配置`hibernate.show_sql=true`：打印执行的SQL
2. 配置`hibernate.format_sql=true`：让打印的SQL可读性更佳
3. 配置`hibernate.use_sql_comments=true`：提供更完备的信息
4. 设置`org.hibernate.type`包的日志输出级别为`TRACE`：打印出具体的数据绑定信息

---

## 使用Hibernate Statistics

其实在Hibernate的`org.hibernate.stat`包下还提供了一个用于输出执行时间等性能数据的模块。启用它之后也能够得到一些统计信息。启用的方法很简单，和上面启用SQL打印的类似：

```
persistence.xml方式：
<property name="hibernate.show_sql" value="true"/>

spring boot方式：
spring:
  jpa:
    properties:
      hibernate.generate_statistics: true
```

相应地还需要添加日志的支持，注意这个是`DEBUG`级别的：

```
<logger name="org.hibernate.stat" level="DEBUG" additivity="false">
	<appender-ref ref="CONSOLE" />
</logger>
```

然后在程序运行期间，会针对每个结束的Hibernate Session打印一些统计信息，例如：

```
2016-12-03 18:50:04.632 DEBUG 42468 --- [pool-2-thread-1] o.h.s.internal.ConcurrentStatisticsImpl  : HHH000117: HQL: select ar from AllocateRecord ar where ar.status = 'WAIT_RECEIVING' and ar.deliveryDueTime < ?1, time: 26ms, rows: 0
2016-12-03 18:50:04.684  INFO 42468 --- [pool-2-thread-1] i.StatisticalLoggingSessionEventListener : Session Metrics {
    56094 nanoseconds spent acquiring 1 JDBC connections;
    0 nanoseconds spent releasing 0 JDBC connections;
    86957 nanoseconds spent preparing 1 JDBC statements;
    26164521 nanoseconds spent executing 1 JDBC statements;
    0 nanoseconds spent executing 0 JDBC batches;
    0 nanoseconds spent performing 0 L2C puts;
    0 nanoseconds spent performing 0 L2C hits;
    0 nanoseconds spent performing 0 L2C misses;
    0 nanoseconds spent executing 0 flushes (flushing a total of 0 entities and 0 collections);
    10698 nanoseconds spent executing 1 partial-flushes (flushing a total of 0 entities and 0 collections)
}
```

可以看到统计的信息种类还是比较多的，大概解释一下：

首先是一条汇总信息：
```
2016-12-03 18:50:04.632 DEBUG 42468 --- [pool-2-thread-1] o.h.s.internal.ConcurrentStatisticsImpl  : HHH000117: HQL: select ar from AllocateRecord ar where ar.status = 'WAIT_RECEIVING' and ar.deliveryDueTime < ?1, time: 26ms, rows: 0
```

它记录下了执行的JPQL(由于使用的是Hibernate，这里显示为HQL)和对应的执行时间(毫秒)以及获取的记录条数。

紧接着是这26毫秒的具体时间组成，简要介绍如下：

1. 56094 nanoseconds spent acquiring 1 JDBC connections：获取JDBC连接的数量和时间
2. 0 nanoseconds spent releasing 0 JDBC connections：释放JDBC连接的数量和时间
3. 86957 nanoseconds spent preparing 1 JDBC statements：准备SQL的数量和时间
4. 26164521 nanoseconds spent executing 1 JDBC statements：执行SQL的数量和时间
5. 0 nanoseconds spent executing 0 JDBC batches：执行批处理操作的数量和时间
6. 0 nanoseconds spent performing 0 L2C puts：更新二级缓存的次数和时间
7. 0 nanoseconds spent performing 0 L2C hits：命中二级缓存的次数和时间
8. 0 nanoseconds spent performing 0 L2C misses：未命中二级缓存的次数和时间
9. 0 nanoseconds spent executing 0 flushes (flushing a total of 0 entities and 0 collections)： flush操作的次数和时间
10. 10698 nanoseconds spent executing 1 partial-flushes (flushing a total of 0 entities and 0 collections)：partial flush操作的次数和时间

需要留意的是使用的时间度量单位是纳秒，也就是说数字看的会比较大，1000000ns才是1ms，不要被这么大的数字吓到了。

有了这么详尽的性能数据，哪里是性能瓶颈就是一清二楚的了。根据这些数据，我们就可以做到有的放矢。对于性能这个问题，一般而言它也是满足帕累托法则的，即解决掉20%的关键问题就可以提升80%的性能，所以说只要明白了性能瓶颈的所在，花费最小的代价提升最大的性能并不是不可能。

---

## 总结

以上列举了两则比较简单实用的用于分析和定位性能瓶颈的方案，都不需要借助多么复杂的工具，只需要使用`Hibernate`自身提供的一些配置和功能即可做到，而且效果也还不错，该有的信息都有了。

如果大家有什么更好的，更神奇的用于定位JPA性能问题的方案，也可以给我留言或者发邮件，欢迎大家一起探讨。

