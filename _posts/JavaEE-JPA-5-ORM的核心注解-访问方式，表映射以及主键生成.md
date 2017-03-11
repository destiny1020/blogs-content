---
title: '[JavaEE - JPA] 5. ORM的核心注解 - 访问方式，表映射以及主键生成'
date: 2016-09-28 23:40:00
categories: ['工具, 库与框架', JPA]
tags: [JavaEE, JPA, ORM]
---

从本篇文章开始，会系统性地介绍JPA中用来实现对象关系映射(Object Relational Mapping)的核心注解，以及基础类型，关系类型，嵌套类型以及集合类型的映射方式。

## 注解种类

在探讨实现JPA中各种映射的方式之前，可以先看看JPA中的注解类型。
由于ORM这一机制涉及到了两个方面：对象(内存模型)以关系数据(关系型数据库)。而显然我们在配置ORM的各种规则时，只能在Java程序中完成。数据库是不知道有JPA这种机制存在的，数据库只是单纯的执行输入的各种SQL语句而已。

因此，我们可以将JPA中的注解笼统地分为两种类型：

1. 逻辑关系注解(Logical Annotation)
2. 物理关系注解(Physical Annotation)

<!-- More -->

所谓的逻辑关系注解，就是声明ORM中的O方，也就是对象实体模型的一些元数据。比如`@Basic`注解，它能够规定基本类型的加载方式，即正常加载(Eager)还是懒加载(Lazy)。
所谓的物理关系注解，就是声明ORM中的R方，也就是关系模型中的一些元数据。比如`@Column`注解，它能够规定一个字段应该被映射到数据库表中的哪个列。

## 访问实体状态的方式

只有实体类型(被标注为`@Entity`的Java类)才能够通过JPA和数据库中的记录进行互相转换。那么在转换的过程中，JPA的实现必然需要对实体类型进行操作。比如在将实体保存到数据库中的时候，JPA的实现需要通过某种方式访问到实体的状态；在将数据库中查询得到的行记录转换成实体对象的时候，JPA的实现需要设置实体的状态。

在JPA中，两种方式完成实体状态的访问：

1. 通过Field
2. 通过Property(也就是Getter/Setter)

### 通过Field

采用这种方式，JPA实现会通过反射的方法来对实体对象的字段进行操作。即使你提供了Getter/Setter，也不会使用它们。那么如何定义试用这种方式呢？通过`@Id`注解：

```java
@Entity
public class Person {
	@Id 
	private int id;
	private String name;
}
```

上述代码中，将`@Id`放在了Field之上。JPA实现根据这一点决定采用基于Field来访问实体状态的方式。

### 通过Property(Getter/Setter)

采用这种方式，JPA时限会通过调用Getter/Setter来对实体对象的字段进行操作。还是一样，通过`@Id`注解来定义：

```java
@Entity
public class Person {
	private int id;
	private String name;

	@Id 
	public int getId() { return id; }
	public void setId(int id) { this.id = id; }
	public String getNickname() { return nickname; }
	public void setNickname(String nickname) { this.name = nickname; }
}
```

此时将`@Id`放在了Getter上。另外，还注意到在访问name字段的时候，使用的Getter和Setter的方法名称分别是`getNickname`以及`SetNickname`。因此在数据库表中使用的列名实际上是nickname，而非实体类中的字段名称name。但是这并不是说在基于Field的访问方式中不能做到字段名和列名不同，通过`@Column`显式定义即可。

### 混合模式

如果混合使用了Field和Property的访问方式，就是所谓的混合模式。通过`@Access`注解进行指定，这是JPA2.0之后添加的新功能。

利用它，可以完成一些从数据表列到实体字段之间基本的转换操作。比如下面这个例子：

```java
@Entity
@Access(AccessType.FIELD)
public class Employee {

	public static final String LOCAL_AREA_CODE = "613";
	@Id 
	private int id;
	@Transient 
	private String phoneNum;

	public int getId() { return id; }
	public void setId(int id) { this.id = id; }
	public String getPhoneNumber() { return phoneNum; }
	public void setPhoneNumber(String num) { this.phoneNum = num; }

	@Access(AccessType.PROPERTY) 
	@Column(name="PHONE")
	protected String getPhoneNumberForDb() {
		if (phoneNum.length() == 10)
			return phoneNum;
		else
			return LOCAL_AREA_CODE + phoneNum;
	}
	protected void setPhoneNumberForDb(String num) {
		if (num.startsWith(LOCAL_AREA_CODE))
			phoneNum = num.substring(3);
		else
			phoneNum = num;
		}
	}

}
```

通过上述代码来讨论一下使用混合访问模式的几个条件：

1. 在类声明上使用`@Access(AccessType.FIELD)`来表明此类是基于字段的访问方式。
2. 在Getter方法使用`@Access(AccessType.PROPERTY)`来表明此字段是基于Getter的访问方式。

在上面的代码中，实现了一些实体字段和数据库列的简单转换。比如，在上述代码中：

```java
@Transient 
private String phoneNum;

@Access(AccessType.PROPERTY) 
@Column(name="PHONE")
protected String getPhoneNumberForDb() {
	if (phoneNum.length() == 10)
		return phoneNum;
	else
		return LOCAL_AREA_CODE + phoneNum;
}
protected void setPhoneNumberForDb(String num) {
	if (num.startsWith(LOCAL_AREA_CODE))
		phoneNum = num.substring(3);
	else
		phoneNum = num;
	}
}
```

在数据库中保存的列名为PHONE。并且在Getter和Setter中都实现了一些简单的转换逻辑。在实体类中，实际用来保存数据的字段名为`phoneNum`。而为了防止该字段也被映射到数据库表中，还特意使用了`@Transient`来表明此字段应该被JPA实现所忽略。

## 表映射

为完成实体类型到数据库表结构的映射，最少只需要使用两个注解：

1. `@Entity`
2. `@Id`

所有的其它配置都会使用默认值。比如数据库表名，如果不指定的话，默认使用的就是实体类型的名称，比如上面的Employee类型，映射的数据库表名即为Employee。

如果需要指定表名，则需要使用`@Table`注解，`@Table`除了支持表名的指定外，还支持schema以及catalog的指定。当然，每种数据库对它们的支持都有所不同。下面列举了几种主流数据库的支持情况：

|数据库|Catalog支持|Schema支持|
|:----    |:---|:----- |
|Oracle |否  |Oracle User ID |
|MySQL |否  |数据库名 |
|SQL Server |数据库名  |对象属主名，2005版开始有变 |
|DB2 |指定数据库对象时，Catalog部分省略  |Catalog属主名 |
|Sybase |数据库名 |数据库属主名 |

所以，即使JPA规范中能够支持catalog和schema，但是由于每种数据库的实现都有所不同，因此在应用中如果依赖它们，会造成应用和某种具体的数据库绑定在一起。幸运的是，需要应用能够自由灵活地在多种数据库之间切换这种需求并不会十分常见。

比如，如果你的应用需要访问两个数据库，那么在使用MySQL时，就能够通过指定schema来实现。当然，这两个数据库还是需要使用相同的连接字符串。

## 主键映射以及生成策略

首先，主键在数据库中的列名可以通过`@Column`进行指定。
主键的数据类型有一些限制，我们通常使用长整型(long)来表示主键。但是除此之外，下面的数据类型也是可以的：

- 简单类型：byte，int，short，long，char
- 相应的包装类型：Byte，Integer，Short，Long，Character
- 字符串类型：String
- 时间类型：Date

主键的生成策略通过`@GeneratedValue`这个注解来声明。总的来说，可以分为下面两种类型：

1. 预先分配
2. 持久化时分配

具体而言，又分为了下面四种类型：

1. AUTO
2. TABLE
3. SEQUENCE
4. IDENTITY

这几种类型都作为`@GeneratedValue`注解中strategy属性的值，比如：

```java
@Id
@GeneratedValue(strategy = GenerationType.AUTO)
private Long id;
```

### 生成策略之AUTO

这是默认的生成策略。当应用程序不关心主键要如何生成，只需要确保能够拥有主键时，可以考虑使用这种方式。因此当JPA的实现遇到这种主键生成策略时，会自行选择使用哪一种。即TABLE，SEQUENCE以及IDENTITY中的一种。

### 生成策略之TABLE

这种方式旨在提供一种和具体数据库无关的主键生成策略。顾名思义，这种方式使用一张数据库表来完成主键的生成。这张表中应该有两列，一列的类型为字符串，表明生成器的名字，它也是这张表中的主键。另一列的类型应该是一个整型，它用来记录最后分配过的数值：

```java
@TableGenerator(name = "Emp_Gen", table = "ID_GEN", pkColumnName = "GEN_NAME", valueColumnName = "GEN_VAL")
@Id
@GeneratedValue(strategy = GenerationType.TABLE, generator = "Emp_Gen")
private Long id;
```

上述代码，声明了用于生成主键的表名为ID_GEN，表中的GEN_NAME列用来保存生成器的名字，即上述Emp_Gen，GEN_VAL用来保存最后分配过的主键值。

因此，可以推断出ID_GEN的结构可以是这样的：

|GEN_NAME|GEN_VAL|
|:----    |:---|
|Emp_Gen |0  |

上述Emp_Gen生成器的GEN_VAL为0表示当前还没有分配过主键。如果在生成这个表的事后希望给GEN_VAL一个初始值，还可以试用initialValue属性进行指定。

为了避免频繁更新ID_GEN表中的记录，可以试用allocationSize来声明需要预分配多少个主键。它的默认值是50，这表示应用程序可以最多分配50个主键后才需要再次访问和更新ID_GEN表。

比如下面的定义：

```java
@TableGenerator(name="Address_Gen", table="ID_GEN", pkColumnName="GEN_NAME", valueColumnName="GEN_VAL", pkColumnValue="Addr_Gen", initialValue=10000, allocationSize=100)
@Id 
@GeneratedValue(generator="Address_Gen")
private int id;
```

就是一份完整的声明。此时的ID_GEN表的结构如下：

|GEN_NAME|GEN_VAL|
|:----    |:---|
|Emp_Gen |0  |
|Addr_Gen |10000  |

另外，如果没有使用结构自动生成(Schema Generation)这一特性，那么还需要保证该表的存在：

```sql
CREATE TABLE id_gen (
    gen_name VARCHAR(80),
    gen_val INTEGER,
    CONSTRAINT pk_id_gen
    PRIMARY KEY (gen_name)
);
INSERT INTO id_gen (gen_name, gen_val) VALUES ('Emp_Gen', 0);
INSERT INTO id_gen (gen_name, gen_val) VALUES ('Addr_Gen', 10000);
```

### 生成策略之SEQUENCE

和使用TABLE的生成策略类似，最好指定清楚所使用的SEQUENCE的名字：

```java
@SequenceGenerator(name="Emp_Gen", sequenceName="Emp_Seq")
@Id 
@GeneratedValue(generator="Emp_Gen")
private int id;
```

同样地，在没有开启结构自动生成时，需要保证序列是存在的：

```sql
CREATE SEQUENCE Emp_Seq
	MINVALUE 1
	START WITH 1
	INCREMENT BY 50
```

需要注意的是并不是所有的数据库都支持SEQUENCE这一特性，目前支持该特性的数据库有： Oracle、PostgreSQL、DB2。

### 生成策略之IDENTITY

常使用MySQL的开发人员肯定知道自增列这一特性，这其实就是IDENTITY这种生成策略所依赖的关键特性。这种策略并非像TABLE和SEQUENCE那样会预分配一段可用的主键，它生成的主键只有在数据Commit之后才会可见。因此在采用该生成策略时，对于未被托管的实体对象而言，其主键通常都是不可用的。JPA实现需要在完成实体的持久化之后再次读取该记录，以此来获取被数据库分配的主键信息。该策略的配置方法如下所示：

```java
@Id 
@GeneratedValue(strategy=GenerationType.IDENTITY)
private int id;
```

需要注意的是并不是所有的数据库都支持IDENTITY这一特性，目前支持该特性的数据库有： MySQL, SQL Server, DB2, Derby, Sybase, PostgreSQL。

