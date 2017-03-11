---
title: '[JavaEE - JPA] 性能优化: 4种触发懒加载的方式'
date: 2016-11-27 23:16:00
categories: ['工具, 库与框架', JPA]
tags: [JavaEE, JPA, Performance, 优化]
---

在一个JPA应用中，可以通过懒加载来提高应用的性能。这一点毋庸置疑，但是懒加载不等于不加载，在某个时刻还是需要加载这些数据的，那么如何触发这个加载的行为才能够事半功倍呢？

这里我想说一点题外话，面试的时候我也会考察被面试者对于JPA/Hibernate的看法，得到的答复通常都包含了对JPA/Hibernate的一些"鄙夷"，比如JPA/Hibernate性能太菜了，现在主流的持久层框架是MyBatis云云。然后我也会试图让他们去分析一下为什么会慢，一部分人不知道如何回答，答曰感觉很慢；另外一些人可能使用的经验稍微多一些，于是答道JPA Provider(例如Hibernate)会生成非常多效率低下的SQL，于是看起来性能就不行了。如果再深究下去问如何改进这些效率低下的SQL呢？能够谈出个一二三的就更是凤毛麟角了。所以我觉得，很多人其实在没有深入调研JPA之前就给出了一个不是很恰当的结论。每种技术都有自身的优缺点，完美的技术是不存在的。具体问题具体分析，不要人云亦云是一个开发人员应该拥有的基本能力。当然我也不是JPA的卫道士，JPA在目前互联网海量数据的环境下，确实有很多的问题，最典型的比如对于数据分片，分表分库上支持的欠缺。我想正是这一点才让MyBatis成为了目前的互联网公司的主流选择。但是，并不是所有的应用都有那么大的数据量，也不是所有的项目都需要去分表分库。更多的中小型项目，如果能够合理地运用好JPA，开发效率和项目的服务性能绝对不会差。毕竟，JPA作为JavaEE标准的一部分，岂能浪得虚名？

所以，这篇文章我想从触发懒加载这个角度，分析几种不同的实现方式，来看看应该如何提高应用的性能。

<!-- More -->

## 数据关联关系的假设

在具体分析4种触发方式之前，我们先来假设一组关联关系：

```java
@Entity
public class Department {
	// 主键等字段
	// ......

	@OneToMany(mappedBy = "department")
	private List<Employee> employees;
}
```

## 通过方法调用触发

这是使用频率最高的一种触发方式，几乎所有JPA开发人员一般情况下都会使用这种方式。甚至在任何场合下都使用这种方式的开发人员也不在少数。

顾名思义，这种方式通过在employee集合对象上调用方法来完成触发，比如下面的代码：

```java
Department dept = em.find(Department.class, deptId);
int count = dept.getEmployees().size();
// ......
```

通过调用size方法来触发懒加载，这个size的执行会让JPA的Provider生成具体去获取集合数据的SQL并执行之。这种方法看似没什么问题，在很多场景下确实也非常好用。但是它太简单粗暴了。在下面两种情况下，都会造成较为严重的性能问题：

1. 集合数据量大。比如关联数据有上成百上千条记录时。
2. 一个实体类型需要触发懒加载的关系很多。比如当上述Department类型还需要加载更多一对多的关系时。

第一种情况很好理解，数据量越大，SQL执行的时间越久，这一点毫无疑问。
第二种情况，假设Department类型有10个一对多关系，现在都需要触发懒加载行为来得到完整的数据。那么针对每个关系都会产生一条SQL命令。加上它自身的，一共就是11条命令。当然你的应用往往不会只有一个用户在使用，假设有100个用户同时使用，那么就是1100条SQL会被执行！这能不慢吗？

所以对于这种触发方式，在确定不会发生上述两种情况时，是可以使用的。一旦有发生它们的风险，就不要使用了。

## 通过Join Fetch触发

这种方式通过在JPQL中添加JOIN FETCH来完成关联关系的获取：

```java
Query q = em.createQuery("SELECT d FROM Department d JOIN FETCH d.employees e WHERE d.id = :id");
q.setParameter("id", deptId);
Department dept = (Department) q.getSingleResult();
```

这种方式，主要解决了在通过方法调用触发时面临的第二个问题：执行的SQL命令数量过多。
使用JOIN FETCH时，执行的SQL命令只有一条。因此，需要触发加载行为的关系越多，使用JOIN FETCH带来的性能优势就越明显。

但是这种方式并非一本万利，如果不同业务场景下需要触发加载的关系不一样，就会产生非常多的组合。而每种组合的JPQL都是不一样的。此时可以结合实际的业务需求通过字符串的拼接操作完成JPQL的准备工作。而这个准备工作在组合情况很多的情况下，往往会十分复杂。不过相比它能够带来的性能提升，这些麻烦都是可以克服的。

除了直接提供JPQL，在Criteria API中也能使用：

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery q = cb.createQuery(Department.class);
Root d = q.from(Department.class);
d.fetch("employees", JoinType.INNER);
q.select(d);
q.where(cb.equal(d.get("id"), deptId));
Department dept = (Department)em.createQuery(q).getSingleResult();
```

这种方式只是定义查询的方式不同而已，在性能层面上和直接写JPQL是一样的。

## 通过NamedEntityGraph触发

这种方式实际上是JPA 2.1中新增的一项特性。借助它也能够完成懒加载的触发。我们先来看看如何定义一个命名的EntityGraph，即NamedEntityGraph。从命名方式上看，是不是很接近于NamedEntityQuery？所以引申到定义方式而言，它们也是很接近的：

```java
@Entity
@NamedEntityGraph(name = "graph.Department.employees", 
      attributeNodes = @NamedAttributeNode("employees"))
public class Department {
	// ......
}
```

使用它进行关系的加载：

```java
EntityGraph graph = em.getEntityGraph("graph.Department.employees");
Map<String, Object> props = new HashMap<>();
props.put("javax.persistence.fetchgraph", graph);
Department dept = em.find(Department.class, deptId, props);
```

此时查询得到的Department对象就包含了我们需要的Employee集合。同样地，使用这种方式的时候也只会生成并执行一条SQL命令。

但是在组合情况比较多的时候，和使用Join Fetch一样，也是需要根据业务场景进行一些准备工作的，只不过这个准备工作更加麻烦，每个组合都需要添加一个专门的@NamedEntityGraph注解用来定义。所以，在组合关系很多的时候，使用@NamedEntityGraph是很不划算的。因此也就有了下面的动态EntityGraph。

## 通过动态的EntityGraph触发

动态EntityGraph的定义方式更加灵活：

```java
EntityGraph graph = this.em.createEntityGraph(Department.class);
Subgraph employeesGraph = graph.addSubgraph("employees");
Map<String, Object> props = new HashMap<>();
props.put("javax.persistence.loadgraph", graph);
Department dept = em.find(Department.class, deptId, props);
```

动态EntityGraph根据需要加载的关系，通过addSubgraph方法进行指定。

## EntityGraph与JOIN FETCH是一样的？

细心的同学看到这里，也许会发现目前介绍的所谓EntityGraph，怎么跟JOIN FETCH那么那么像呢？难道只是换了个马甲？

很显然，并没有这么简单。

注意一下上面的这两行代码：

```java
// #1 loadgraph
props.put("javax.persistence.loadgraph", graph);

// #2 fetchgraph
props.put("javax.persistence.fetchgraph", graph);
```

这两者有什么具体的区别呢？这里我不打算长篇累牍地介绍。捡重点说就是：

1. loadgraph：在原有Entity的定义的基础上，定义**还**需要获取什么字段/关系
2. fetchgraph：完全放弃原有Entity的定义，定义**仅**需要获取什么字段/关系

注意上面的"还"和"仅"，表达了两者最大的不同点。

举个例子，如果我们的Department类型中还有一个name字段：

1. loadgraph：被加载的数据为name以及employees
2. fetchgraph：被加载的数据仅为employees

所以，在使用EntityGraph的时候配合fetchgraph，可以精准的完成所需要数据的加载，可谓是指哪打哪。在一些对性能尤其敏感的业务场景，不妨来试试看仅仅加载所需要的数据的那种酸爽吧。

关于此话题的更多资源，可以参考[官方文档](https://docs.oracle.com/javaee/7/tutorial/persistence-entitygraphs001.htm)

---

## 结语

分析比较了以上这4种用于触发懒加载的方式，我们应该有能力辨别出在什么场景下使用哪种方式更适合了。还不快去在你的工程中尝尝鲜？



