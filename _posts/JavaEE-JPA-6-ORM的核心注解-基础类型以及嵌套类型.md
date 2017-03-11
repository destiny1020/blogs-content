---
title: '[JavaEE - JPA] 6. ORM的核心注解 - 基础类型以及嵌套类型'
date: 2016-10-17 22:26:00
categories: ['工具, 库与框架', JPA]
tags: [JavaEE, JPA, ORM, 注解]
---

本文继续介绍JPA ORM的核心注解中和基础类型映射相关的部分。

## 基础类型映射

所谓的基础类型映射，实际上就是Java中定义的数据类型应该如何被JDBC转换成数据库所支持的数据类型。而这些基础类型，主要包括了以下9种：

1. 简单类型：byte，int，short，long，boolean，char，float以及double
2. 简单类型对应的包装类型：Byte，Integer，Short，Long，Boolean，Character，Float以及Double
3. 字节以及字符数组：byte[]，Byte[]，char[]以及Character[]
4. 能够表示大数值的类型：java.math.BigInteger以及java.math.BigDecimal
5. 字符串类型：String
6. Java时间类型：java.util.Date以及java.util.Calendar
7. JDBC时间类型：java.sql.Date，java.sql.Time以及java.sql.Timestamp
8. 枚举类型
9. 可序列化的对象(Serializable Object)

<!-- More -->

在Java源代码中，可以使用`@Basic`来标明某个属性是需要被持久化的。但是这个注解一般而言是可选的，出现在实体类型中的属性默认就是需要被持久化的。而正是因为`@Basic`注解只能够应用在以上列举的9种类型之上，所以我们将这些类型命名为基础类型，同Basic这个词的意思。

### 列映射(Column Mapping)

如果说`@Basic`是一种逻辑映射(Logical Annotation)的话，那么与之相对的`@Column`便是物理映射(Physical Annotation)。它能够规定属性应该如何被映射成数据库表中的列。一般最常用的就是其中的name属性，它能够规定将数据转换到数据库表中后列的名字。但是，这个注解支持的属性远不止一个name，还有很多：

![这里写图片描述](http://img.blog.csdn.net/20160928172755827)

限于篇幅，就不一一列举了。在需要的时候查阅一下即可。

### 懒加载(Lazy Fetching)

一般说起懒加载，说的都是在集合映射的一对多或者多对多关系下的懒加载。但是对于基础类型，也是支持懒加载的。这一点恐怕让很多人觉得有点匪夷所思，但是考虑当一个属性的数据量非常巨大的时候(比如下面即将提到的大对象)，懒加载还是有必要的。

```java
@Basic(fetch=FetchType.LAZY)
@Column
private String article;
```

比如上述的article字段的数据量可能特别大，因此使用了懒加载。通过指定`@Basic`注解上的fetch属性来实现。但是需要注意的是，懒加载的声明对于JPA实现只是一个提示(Hint)，表明应用程序希望使用懒加载，但是JPA实现是否真的会实现还取决于它自身是如何实现的。而且，特别注意并不是所有的场合都适合使用懒加载，如果对一个普通的字符串属性使用懒加载，JPA实现在处理它的时候，还需要额外进行一堆操作，比如为该属性加上代理，当属性被访问的时候由代理发出加载数据的请求等等。这些操作都需要消耗时间，内存等资源。因此，盲目地使用懒加载只会让程序的性能变差。

### 大对象(Large Object，LOB)

对于JDBC而言，处理拥有大数据量的大对象的方式和处理其它普通对象的方式是有所不同的。因此为了通知JPA实现某个属性是大对象，就需要使用`@Lob`注解。结合`@Basic`的懒加载，可以声明如下：

```java
@Entity
public class Employee {
	@Id
	private int id;

	@Basic(fetch=FetchType.LAZY)
	@Lob 
	@Column(name="PIC")
	private byte[] picture;
	// ...
}
```

`@Lob`注解本身并不包含任何属性，它的作用正如上面所讨论的那样，只是为了某种标注来通知JPA实现。此类注解也可以被称为标记注解(Marker Annotation)。

### 枚举类型(Enumerated Type)

我们都知道，枚举类型中有两个比较常用的方法，一个是`ordinal()`，一个是`name()`。它们的作用分别是得到枚举值的声明顺序和获取枚举值的声明名称。当枚举类型被映射到数据库表中的时候，默认使用的是`ordinal()`方法的返回值。如果需要使用name()方法的返回值作为映射后的值，可以使用下面的方式声明：

```java
@Enumerated(EnumType.STRING)
private EmployeeType type;
```

`EnumType`还有另一个名为ORDINAL的取值，它是默认值。

那么，对于枚举类型的映射，一般使用哪种比较好呢？通常而言，使用ORDINAL会更好一些(也就是使用默认选项)。因为这样的存储效率会更高一些，毕竟存储整型值比存储字符串类型的代价会小一些。但是需要注意的是，如果随着应用的功能越来越多，枚举类型的候选值也可能会越来越多，这个时候为了保证以前存储的值的有效性，需要注意新追加的枚举值总是需要被声明为最后一个，从而不会影响到前面已有枚举值的ORDINAL。

### 时间类型(Temporal Type)

谈到时间类型，参与到映射的时间类型实际上有两种：

1. Java时间类型：java.util.Date以及java.util.Calendar
2. JDBC时间类型：java.sql.Date，java.sql.Time以及java.sql.Timestamp

对于第二种JDBC的时间类型，不需要任何注解就能完成正确的映射和转换。
对于第一种Java的时间类型，就需要使用一些注解了。主要是为了告知JPA实现在映射和转换时使用哪一种JDBC时间类型(因此，最终还是使用的JDBC时间类型)。通过`@Temporal`注解来指定：

```java
@Temporal(TemporalType.DATE)
private Calendar dob;

@Temporal(TemporalType.TIMESTAMP)
private Date startTime;
```

TemporalType枚举类型的可选枚举值如下所示：

![这里写图片描述](http://img.blog.csdn.net/20160928213624986)

### 瞬态属性(Transient State Attribute)

如果一个属性它属于某个实体类型，但是在持久化到数据库中的时候，又不需要将它也映射到表结构中，就可以将它设置成"瞬态属性"。

可以通过两种方式实现：

1. 使用`transient`关键字
2. 使用`@Transient`注解

至于它俩的区别？那要从transient这个关键字的其它用法说起了。我们都知道transient是Java语言中的一个关键字，它并非为JPA而设计。该关键字最典型意义是当一个对象参与到序列化的过程中时，被transient修饰的字段是不会参与序列化的。知道了这一点，这两种实现方式的区别也就很明确了：被`@Transient`注解修饰的属性还是会正常参与到序列化过程中，但是被transient关键字修饰的就不会了。

## 嵌套类型(Embedded Type)

介绍完了基础类型，再来看看嵌套类型。嵌套类型听起来很高大上的样子，但是实际上你完全可以把它当作几个属性(特别是基础类型)的集合，而这几个属性在逻辑上一般有比较紧密的关系。但是在实际的物理存储中(数据库中)，嵌套类型的字段还是会和其归属实体一起被存储到一张表中。

从上面讨论的特点来看，可以将嵌套类型视为一种独立性不那么强的实体类型，它总是需要依赖于另一个实体类型，不能单独存在。举个例子，地址这种实体概念就非常适合被定义为一个嵌套类型，比如家庭地址，收件地址等等，都拥有共通的几个字段，但是它们通常是属于另外一个实体类型，比如客户这一实体，它就能够拥有家庭地址以及收件地址。

如果从UML(统一建模语言)来审视这个概念的话，通常适用于使用嵌套类型来表达的实体类型一般都体现在它能够被"组合(Composition)"到另外一个实体中：

![这里写图片描述](http://img.blog.csdn.net/20160928223406929)

解释清除了嵌套类型的存在意义，下面来看看如何声明和使用它。

声明嵌套类型Address：

```java
@Embeddable
public class Address {
	private String street;
	private String city;
	// ...
}
```

使用嵌套类型Address：

```java
@Entity
public class Employee {
	@Id 
	private int id;
	private String name;
	private long salary;
	@Embedded 
	private Address address;
	// ...
}

@Entity
public class Company {
	@Id 
	private int id;
	private String name;
	@Embedded 
	private Address address;
	// ...
}
```

注意`@Embeddable`和`@Embedded`这两个注解分别用来声明和使用嵌套类型，不要弄混了。

在上述代码中，Employee和Company两个实体类型都使用Address嵌套类型。因此它们两个实体类型所对应的数据库表结构中都会存在Address类型定义的几个属性。

那么有没有办法修改嵌套类型被映射到表后的属性名称呢？
当然也是可以的，通过`@AttributeOverrides`和`@AttributeOverride`这两个注解来实现这一需求：

比如，在Employee类型中，嵌套类型Address的city字段所对应的数据库列名需要被设置成province，street需要被设置成area。可以使用如下的代码实现：

```java
@Entity
public class Company {
	@Id 
	private int id;
	private String name;
	@Embedded
	@AttributeOverrides({
		@AttributeOverride(name="street", column=@Column(name="area")),
		@AttributeOverride(name="city", column=@Column(name="province"))
	})
	private Address address;
	// ...
}
```

