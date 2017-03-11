---
title: '[JavaEE - JPA] 7. ORM的核心注解 - 关系类型'
date: 2016-10-20 21:40:00
categories: ['工具, 库与框架', JPA]
tags: [JavaEE, JPA, ORM, 注解]
---

本文继续介绍JPA ORM的核心注解中和关系映射相关的部分。

关系映射的处理绝对是一个JPA应用最为重要的部分之一。关系映射处理的好，不仅仅是建模上的成功，而且在程序性能上也会更胜一筹。关系映射处理的不好很容易造成程序性能底下，各种Bug频繁出现，而且这些Bug通常还会比较隐蔽，总是在关键时刻掉链子。我想这也是为什么很多开发人员说JPA入门容易，精通难得原因之一。因为关系确实不是那么好处理的，不仅需要对业务有相当深刻的见解，更需要对JPA提供的各种关系映射类型有入木三分的理解。

本文就尝试来理一理JPA中的各种关系映射类型。

<!-- More -->

## 关系的基本术语

在介绍JPA提供的几种关系映射类型之前，有必要先来学习一下关于关系的三个基本术语：角色，方向和基数。这对于理解关系的本质十分有帮助。

### 角色(Role)

所谓"一个巴掌拍不响"，一个关系不可能只有一个参与方，而且任何由多个参与方组成的关系必定都可以拆解成两两关系。因此，在这里我们也只考虑由两个参与方所组成的关系。比如我们常见的雇佣关系，就是企业和员工之间的一种关系，那么企业和员工在这层关系中就分别扮演着雇佣者和被雇佣者的角色。而且在现实生活中，一个人是可以同时扮演者多种角色的，比如被雇佣者在家庭中可以作为妻子/丈夫/孩子/父亲/母亲等角色，反映到程序中就是一个实体可以被别的实体所引用，一旦被引用，即代表了关系的建立。被引用的次数越多，那么就表示这个实体所承担的角色就越多。这一点很好理解，比如当Employee实体在Department实体中被引用，表示Employee承担的是部门员工的角色；当Employee实体在Payroll实体中被引用时，就表示Employee此时承担的是薪酬领取者的角色。

### 方向(Directionality)

除了角色之外，方向是关系的另一个要素。关系不会毫无缘由的诞生，总需要有一个角色来打破僵局，建立这层关系。那么主动建立的一方我们可以将其称为源角色(Source Role，简称Source)，而被动响应的一方则可以被称为目标角色(Target Role，简称Target)。

反映到程序中，关系的方向指的就是主动引用，比如我们在Employee类型中引用Department类型，那么就是由Employee指向Department的一层关系，Employee扮演的是Source，Department扮演的是Target。如果在Department类型中也引用了Employee类型，那么就是由Department指向Employee的一层关系，Department扮演的是Source，Employee扮演的是Target。

因此如果互相引用，那关系就是一个双向关系了。就好比我知道我爸爸是谁，我爸爸也知道我是谁。

而单向关系在这个世界中其实更多一些，比如我知道马云是谁，而马云肯定不知道我是谁。又或者一个男生暗恋一个女生，这些都是单向关系。

### 基数(Cardinality)

关系的最后一个要素便是基数。所谓的基数实际上描述的是一个很简单的概念，比如法律上的一夫一妻制和一夫多妻制。即参与到关系的两个角色，在数量上的特征。一个部门可以有多个员工，而一个员工通常只属于一个部门。一个学生可以参与到多个社团，而一个社会也可以拥有多个学生作为其成员。

反映到程序中，就是引用类型是一个单值类型，还是一个集合类型的区别。如果在Employee类型中引用Department类型，通常只会引用声明一个department实体。但是反过来再Department类型中引用其Employee类型的时候，通常会使用一个集合来表示其下所有的Employee。

## 关系映射

基础概念介绍完毕，下面开始进入正题。

首先，根据关系中目标角色的数量，可以将关系简单分为两种：

1. 单值映射(Single-Valued Mapping)
2. 集合映射(Collection-Viewed Mapping)

### 单值映射

所谓单值映射，就是目标角色的基数(Cardinality)等于1。也就意味着存在两种情况：

#### 一对一(One-to-One)

典型的例子比如，Employee类型(雇员)和Workspace类型(工位)之间的关系。此时使用JPA提供的`@OneToOne`注解进行描述：

```java
@Entity
public class Employee {
	@Id 
	private int id;

	private String name;

	@OneToOne
	@JoinColumn(name="WSPACE_ID")
	private Workspace workspace;
	// ...
}

@Entity
public class Workspace {
	@Id 
	private int id;

	private String location;
}
```

上述代码是一对一单向关系的示例代码。其中出现了一个名为`@JoinColumn`的注解，可以将这个注解理解成外键。而这个外键的列名则是通过`@JoinColumn`注解中的`name`属性进行声明。同时，还需要注意的是当`@OneToOne`和`@JoinColumn`一起使用的时候，这个外键所在的列实际上还需要满足唯一性的约束。因为每个Workspace的实例实际上是被唯一的一个Employee实例所独占的，所以在该外键列中不可能存在相等的值。

那么一对一双向关系如何用JPA提供的注解进行声明呢？比如下面这一段代码，Employee不仅引用了Workspace类型，Workspace中也同时引用了Employee类型：

```java
@Entity
public class Employee {
	@Id 
	private int id;

	private String name;

	@OneToOne
	@JoinColumn(name="WSPACE_ID")
	private Workspace workspace;
	// ...
}

@Entity
public class Workspace {
	@Id 
	private int id;

	private String location;

	@OneToOne(mappedBy = "workspace")
	private Employee employee;
	// ...
}
```

注意在上述的Workspace类中，也使用了`@OneToOne`注解来声明一个从Workspace指向Employee的关系。但是这里并没有使用`@JoinColumn`注解来声明外键的相关信息。也就是说，上述实体类对应的数据库表中并不会含有引用Employee类型的外键列。这是JPA中规定的对于双向一对一关系的映射方式。尤其注意`@OneToOne`注解中的`mappedBy`属性，这个属性的值实际上是关系中源角色(Employee)一方引用目标角色(Workspace)一方的引用名称，加入我们把Employee类中的workspace改成ws，那么相应的Workspace类中`@OneToOne`的`mappedBy`属性也需要被改成ws。这种定义方式，保证了所定义的双向关系是由参与的两个@OneToOne来完成的。

如果不考虑规范的话，更加直观地定义双向一对一的方式应该是这样的：

```java
@Entity
public class Employee {
	@Id 
	private int id;

	private String name;

	@OneToOne
	@JoinColumn(name="WSPACE_ID")
	private Workspace workspace;
	// ...
}

@Entity
public class Workspace {
	@Id 
	private int id;

	private String location;

	@OneToOne
	@JoinColumn(name="E_ID")
	private Employee employee;
	// ...
}
```

这种定义方式当然可以，但是这样定义的两个一对一关系之间就没有了联系，它们实际上定义了两个单向的一对一关系。JPA不会认为这两个一对一关系实际上共同构成了一个双向的一对一关系。体现在表结构上，就是在Workspace对应的表结构中，也会有一个外键列用来引用Employee的相应记录。

因此，如果你希望定义的是一个双向的一对一关系，还是遵守JPA的规范，正确地使用`@OneToOne`注解及其`mappedBy`属性和用于定义外键列的`@JoinColumn`注解吧。

最后补充一点，由于`@JoinColumn`是用来定义外键关系的，那么拥有该注解的一方可以被称为关系中的所有方(Owning Side)，而被引用的一方则被称为反转方(Inverse Side)。所以如果是定义双向关系的话，在反转方一般就需要使用`mappedBy`属性了。

#### 多对一(Many-to-One)

多对一这种关系的例子可谓是随手拈来，比如上文中多次提到的Employee和Department之间的关系，就可以这样描述：

```java
@Entity
public class Employee {
	@Id 
	private int id;

	@ManyToOne
	@JoinColumn(name="DEPT_ID")
	private Department department;
	// ...
}
```

使用`@ManyToOne`注解来定义一个多对一关系。对于多对一关系而言，它一般需要一个`@JoinColumn`来帮助定义其对应的外键信息。因此根据上面的定义，多对一关系中的"多"方，总是可以被视为此段关系中的所有方(Owning Side)，而响应的"一"方，在这种情况下可以被视为反转方(Inverse Side)。因此，在上面的例子中，Employee的表结构中就会有一个名为DEPT_ID的列作为引用Department记录的外键列。

有了"多对一"，相应地就会有"一对多"。这便是我们即将要介绍的关系。

### 集合映射

所谓集合映射，就是目标角色的基数(Cardinality)大于1。也就意味着存在下面两种情况：

#### 一对多(One-to-Many)

一对多这种关系一般是不会单独出现的，比如我们只想定义从Department指向Employee的一层关系。假设一个Department实例中引用了五个Employee实例，在程序中使用集合类型来描述这层关系很方便，也没有什么障碍。但是考虑一下换到数据库表结构中，应该如何描述这种结构呢？显然在Department对应的表中是没法描述的，毕竟一行是无法引用多个(并且数量还不定)其它表结构中的记录的。所以一对多一般伴随着多对一作为双向关系出现，而且由于`@JoinColumn`总是会出现在"多"方，因此我们需要在`@OneToMany`注解中使用`mappedBy`属性来声明这一点，毕竟此时的"一"方式反转方(Inverse Side)，在反转方中就需要使用`mappedBy`属性，这是JPA规范中的一部分。

```java
@Entity
public class Department {
	@Id 
	private int id;
	
	private String name;
	@OneToMany(mappedBy="department")
	private Collection<Employee> employees;
	// ...
}
```

那么是不是一定要使用`mappedBy`来表示这个一对多关系是一个双向关系中的一部分呢？也不一定。
如果你只想定义一个单向的一对多关系，那么可以不使用`mappedBy`属性，此时JPA会推断你想使用联接表(Join Table)来完成关系的定义，虽然这个联接表有其默认的命名规范，但是显式地声明它通常是更明智的方案，比如下面这样：

```java
@Entity
public class Department {
	@Id 
	private int id;

	private String name;

	@OneToMany
	@JoinTable(name="DEPT_EMP",
		joinColumns=@JoinColumn(name="DEPT_ID"),
		inverseJoinColumns=@JoinColumn(name="EMP_ID"))
	private Collection<Employee> employees;
	// ...
}
```

此时，Department和Employee的一对多关系会通过表DEPT_EMP进行管理。`@JoinTable`注解的`joinColumns`属性声明了联接表中引用此关系中的所有方的外键列名为DEPT_ID，`inverseJoinColumns`属性声明了联接表中引用此关系中的反转方的外键名为EMP_ID。

#### 多对多(Many-to-Many)

一个简单的多对多的例子是，一个员工可以工作在多个项目中，而一个项目也有多个员工工作在其中：

```java
@Entity
public class Employee {
	@Id 
	private int id;

	private String name;

	@ManyToMany
	@JoinTable(name="EMP_PROJ",
		joinColumns=@JoinColumn(name="EMP_ID"),
		inverseJoinColumns=@JoinColumn(name="PROJ_ID"))
	private Collection<Project> projects;
	// ...
}

@Entity
public class Project {
	@Id 
	private int id;

	private String name;

	@ManyToMany(mappedBy="projects")
	private Collection<Employee> employees;
	// ...
}
```

很显然，这是一个双向多对多的例子。之所以为双向，主要还是取决于在Project类中声明的`@ManyToMany(mappedBy="projects")`，如果缺少了这个`mappedBy`属性的相关定义，那么JPA实现在分析这段代码中出现的两个`@ManyToMany`的时候，会认为这是两个单向而独立的多对多关系。因此会尝试读取两个联接表：一个是我们显式声明的表EMP_PROJ，另一个则是根据实体类的名字生成的默认联接表Project_Employee。显然生成两个独立的联接表在通常情况下并非我们的目的，所以在使用多对多关系的时候，一定要注意`mappedBy`属性的声明。

