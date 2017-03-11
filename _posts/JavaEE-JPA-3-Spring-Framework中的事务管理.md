---
title: '[JavaEE - JPA] 3. Spring Framework中的事务管理'
date: 2016-09-23 23:05:00
categories: ['工具, 库与框架', JPA]
tags: [JavaEE, JPA, 事务, Spring]
---

前文讨论了事务划分(Transaction Demarcation)在EJB中是如何实现的，本文继续介绍在Spring Framework中是如何完成事务划分的。

我们已经知道了当采用Container事务类型的时候，事务划分主要有以下两种方案(参考[这里](http://blog.csdn.net/dm_vincent/article/details/52566964))：

1. 使用JTA接口在应用中编码完成显式划分
2. 在容器的帮助下完成自动划分

在使用JavaEE的EJB规范时，这两种方案分别被实现为BMT以及CMT，关于BMT和CMT在上一篇文章中有比较详尽的讨论(参考[这里](http://blog.csdn.net/dm_vincent/article/details/52579719))。

那么对于Spring Framework而言，又是如何来实现上述两种方案的呢。

<!-- More -->

Spring Framework的主要优势之一就是屏蔽了底层容器的很多实现细节，它甚至会在JavaEE的标准之上进行一些封装，来最大程度地做到平台无关，容器无关。在使用Spring Framework开发JavaEE应用的时候，对底层容器的依赖并不明显。因此它并没有像EJB那样当采用基于Container的事务类型时，严重地依赖于JTA这一规范。它通过提供下面两种实现方案完成了对于事务的划分：

1. 声明式的事务管理(Declarative Transaction Management)
2. 编程式的事务管理(Programmatic Transaction Management)

虽然名字叫法不同，但是本质是差不多的。

第一种，声明式的事务管理，目的就是要帮助开发人员完成事务的自动划分。这一点和EJB CMT非常类似。第二种，编程式的事务管理，目的就是让开发人员能够有办法自行控制事务应该如何进行。这一点和EJB BMT非常类似。

## Spring Framework事务管理概要

在介绍具体的声明式以及编程式这两种事务管理方案之前，还是需要简单介绍一下Spring Framework是如何进行事务管理的。

在传统的JavaEE中，开发人员一般可以采用两种事务类型：

1. Resource-local 本地事务
2. Global 全局事务 (一般需要应用服务器提供的容器环境，所以也称为Container事务)

对于本地事务而言，它没有办法针对多个事务性资源，往往只能针对单一的事务资源。而且它也过于底层，对于业务逻辑的侵入性很强，这一点写过基于JDBC应用的同学都知道，20行代码里面可能只有5行是业务相关的，剩下15行都用来处理事务和相关异常了。所以这样的代码很难维护，看上去也不那么优雅。

对于全局事务而言，它克服了本地事务的缺点，能够处理多个事务性资源(典型的比如数据库，消息队列等)。但是它依赖于笨重的JTA(Java Transaction API)，这一套和事务相关的API用起来也是让人叫苦不迭，和JDBC类似，都有太过底层的问题。更重要的是，JTA一般是应用服务器才会提供的服务，因此使用全局事务的条件也算是比较苛刻。

所以针对以上的种种痛点，Spring Framework建立了一套关于事务的抽象。让开发人员可以在任何环境中方便地使用事务来管理资源。甚至在一般情况下连应用服务器也不需要了，使用一般的Web容器即可，比如流行的Tomcat(只有在需要同时处理多个事务性资源的情况下，才可能需要使用应用服务器，但是一般的应用显然没有那么复杂)。

### Spring Framework对事务的抽象

Spring Framework通过引入一个名为事务策略(Transaction Strategy)的概念来建立这个抽象。具体而言，表现为下面这个接口：

```java
public interface PlatformTransactionManager {
	TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;
	void commit(TransactionStatus status) throws TransactionException;
	void rollback(TransactionStatus status) throws TransactionException;
}
```

`org.springframework.transaction.PlatformTransactionManager`接口本质上是一个服务提供接口(Service Provider Interface, SPI)。这也算是实现系统扩展性的一种经典设计模式了，具体可以参考[维基百科](https://en.wikipedia.org/wiki/Service_provider_interface)。比如我们所熟知的JDBC就是依赖于这一模式，各种数据库厂商都实现了JDBC SPI中的功能从而使他们的数据库能够和Java应用通信。

接口中定义的每个方法都可能会抛出一个名为`TransactionException`的异常，这个异常不像是JTA的相关接口中定义的那些异常基本上都是受检异常(Checked Exception)，而`TransactionException`是一个运行时异常(Runtime Exception)。这也反映出了Spring Framework的设计原则，尽量不给开发人员添乱。毕竟处理事务相关的异常可不是一门轻松活。一般而言都是直接抛出去，谁有能力处理交给谁吧。所以这里将异常定义为运行时异常默认了开发人员不会处理它，当然如果有能力处理的话加上try...catch语句就好了。

下面简要介绍一下上述接口中的三个方法：

```java
TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;
```

这里有出现了两个新概念。
接受的参数是`TransactionDefinition`接口，返回的对象是`TransactionStatus`接口。

在`TransactionDefinition`接口的文档中，有这么一段话：

> Interface that defines Spring-compliant transaction properties.
> Based on the propagation behavior definitions analogous to EJB CMT attributes.

翻译一下就是：这个接口定义了Spring兼容的事务属性。这些属性类似于EJB CMT中关于事务传播(Transaction Propagation)行为的定义。而这个事务传播实际上就是指的`@TransactionAttribute`中定义的那些个属性，诸如`MANDATORY`，`REQUIRED`，`REQUIRES_NEW`等，详情可以参考[这里](http://blog.csdn.net/dm_vincent/article/details/52579719#t4)。

知道了这一点，再来看看里面定义了些什么：

```java
public interface TransactionDefinition {
    // 传播属性
	int PROPAGATION_REQUIRED = 0;
	int PROPAGATION_SUPPORTS = 1;
	int PROPAGATION_MANDATORY = 2;
	int PROPAGATION_REQUIRES_NEW = 3;
	int PROPAGATION_NOT_SUPPORTED = 4;
	int PROPAGATION_NEVER = 5;
	int PROPAGATION_NESTED = 6;
	// 隔离属性
	int ISOLATION_DEFAULT = -1;
	int ISOLATION_READ_UNCOMMITTED = Connection.TRANSACTION_READ_UNCOMMITTED;
	int ISOLATION_READ_COMMITTED = Connection.TRANSACTION_READ_COMMITTED;
	int ISOLATION_REPEATABLE_READ = Connection.TRANSACTION_REPEATABLE_READ;
	int ISOLATION_SERIALIZABLE = Connection.TRANSACTION_SERIALIZABLE;
	// 超时属性
	int TIMEOUT_DEFAULT = -1;

	// 行为
	int getPropagationBehavior();
	int getIsolationLevel();
	int getTimeout();
	boolean isReadOnly();
	String getName();
}
```

可见，这个接口里面定义了各种属性，同时也定义了获取一个事务属性的方法。所以将该接口作为上述`getTransaction`方法的参数，目的就很清晰了：根据所描述事务的属性来获取具体的事务。就好比传入一个config对象，得到一个符合该config定义的对象那样。

然后，方法返回的`TransactionStatus`又是啥呢：

```java
public interface TransactionStatus extends SavepointManager, Flushable {
	boolean isNewTransaction();
	boolean hasSavepoint();
	void setRollbackOnly();
	boolean isRollbackOnly();
	void flush();
	boolean isCompleted();
}
```

这就很明显了，它和之前介绍过的JTA提供用于实现BMT的`UserTransaction`相似度很高。它代表的就是一个当前执行线程所关联的一个具体事务对象。根据`TransactionDefinition`所定义的事务属性，这个具体的事务对象可以是一个刚刚创建的(当传播属性定义为`PROPAGATION_REQUIRES_NEW`时)，也可以是复用的一个事务对象(当传播属性定义为`PROPAGATION_SUPPORTS`或者`PROPAGATION_REQUIRED`并且确实运行在事务环境中时)。

PlatformTransactionManager接口中剩下的两个方法`commit()`和`rollback()`看名字就知道它们怎么用了，因此不再一一介绍了。

PlatformTransactionManager接口的实现是需要根据具体的环境而被注入到运行时环境中的，比如JTA，JDBC，Hibernate等。关于具体的注入方法可以参考Spring Framework的相关文档，这里就不再细说。

## 声明式的事务管理(Declarative Transaction Management)

值得一体的是，Spring Framework是通过AOP(面向切面编程，Aspect-Oriented Programming)技术来实现声明式的事务管理的。关于AOP，如果想要了解更多可以参考[Wiki](https://en.wikipedia.org/wiki/Aspect-oriented_programming)。

其实Spring Framework在实现声明式的事务管理时，多少也借鉴了一些EJB CMT的实现方式，取其精华去其糟粕。那么将它和EJB CMT比较的话，主要有以下几个方面的提高：

1. 不再像EJB CMT那样局限于JTA。Spring Framework提供的声明式事务管理能够运行在几乎任何主流环境下，比如JTA，JDBC，Hibernate等。开发人员所需要做的只是根据底层环境提供相应的配置即可。
2. 可以对任何类使用声明式事务管理，而不像EJB CMT那样只能对有限的几种类型，比如`@Stateful`，`@Stateless`等会话Bean。
3. 声明式的回滚规则，比如指定抛出了哪些类型的异常时才发生回滚。这一特性是EJB CMT中不存在的。
4. 通过AOP技术定制化事务行为，比如在事务发生回滚的时候执行自定义代码。而在EJB CMT中对于回滚你能做的仅仅是调用`setRollbackOnly()`。

和EJB CMT比较相近的一点是，它们都在发生了运行时异常(Runtime Exception)时会触发回滚操作，但是对于受检异常(Checked Exception)是不会主动回滚的，一般的理解是需要开发人员来处理，在catch语句中显式地执行回滚操作。

说的这么好听，那么如何使用声明式的事务管理呢。其实用法很简单，就两个步骤(以注解配置方式为例)：
1. 在`@Configuration`的JavaConfig类中使用`@EnableTransactionManagement`
2. 在需要使用事务的类或者方法上使用`@Transactional`

要想理解它是如何实现的，首先需要了解AOP Proxy这一概念。也就是说，在开发人员声明了`@Transactional`之后，Spring Framework会利用AOP技术生成一个代理对象(Proxy)，这个代理对象使用`TransactionInterceptor`并结合`PlatformTransactionManager`来实现事务的相关功能。

可以用下面这张图来表示：

![这里写图片描述](http://img.blog.csdn.net/20160920000339617)

在Spring Framework没有介入之前，只存在上图中的调用者和目标方法这两个部分。而在介入后利用AOP以及动态代理技术首先为方法所在对象创建了一个代理对象，然后根据配置生成各种AOP任务，如上图中的事务Advisor以及其它类型的Advisor(日志任务等)。然后在真正发生方法调用的时候，调用的顺序如上图中的数字1-8。在事务Advisor被执行的时候(步骤2)才会真正创建事务，然后在步骤4执行的是业务逻辑。随后执行流程开始依次返回，到步骤7发生的时候事务会根据其是否成功而提交或者是回滚。

对于具体的基于XML以及注解的配置方法，可以查看[Spring Framework Transaction部分的相关文档](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#transaction)。

## 编程式的事务管理(Programmatic Transaction Management)

Spring Framework提供了两种方法来支持编程式的事务管理：

1. `TransactionTemplate`
2. `PlatformTransactionManager`

推荐使用第一种方式。`TransactionTemplate`在形式上类似于Spring JDBC中提供的`JdbcTemplate`，封装了很多模板代码，让开发人员可以专注到业务逻辑的开发上。第二种类似于JTA中提供的`UserTransaction`，但是简化了部分异常处理。


### 利用`TransactionTemplate`完成编程式事务管理

下面是一段使用`TransactionTemplate`完成编程式事务管理的代码片段(下面的摘自Spring Framework官方文档)：

```java
public class SimpleService implements Service {
	// 共享的TransactionTemplate实例
	private final TransactionTemplate transactionTemplate;
	// 利用构造器注入将PlatformTransactionManager的实现注入到类中
	public SimpleService(PlatformTransactionManager transactionManager) {
		Assert.notNull(transactionManager, "The 'transactionManager' argument must not be null.");
		this.transactionTemplate = new TransactionTemplate(transactionManager);
	}
	public Object someServiceMethod() {
		return transactionTemplate.execute(new TransactionCallback() {
			// 此方法中的代码会在事务上下文中执行
			public Object doInTransaction(TransactionStatus status) {
				updateOperation1();
				return resultOfUpdateOperation2();
			}
		});
	}
}
```

使用了回调的风格完成了编程式的事务管理，其中比较关键的是`TransactionCallback`类的匿名实现，它作为参数传入到`TransactionTemplate`的execute方法中。

如果事务相关代码并没有返回值，那么可以使用不带参数的`TransactionCallbackWithoutResult`类的匿名实现。

```java
transactionTemplate.execute(new TransactionCallbackWithoutResult() {
	protected void doInTransactionWithoutResult(TransactionStatus status) {
		try {
			updateOperation1();
			updateOperation2();
		} catch (SomeBusinessExeption ex) {
			// 发生异常时，回滚事务
			status.setRollbackOnly();
		}
	}
});
```

### 利用`PlatformTransactionManager`完成编程式事务管理

可以在获取到`PlatformTransactionManager`的实现后通过下面的逻辑完成事务管理：

```java
DefaultTransactionDefinition def = new DefaultTransactionDefinition();
// 只有在编程式事务管理中才能设置事务的名称
def.setName("SomeTxName");
def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
TransactionStatus status = txManager.getTransaction(def);
try {
	// 执行业务逻辑
}
catch (MyException ex) {
	txManager.rollback(status);
	throw ex;
}
txManager.commit(status);
```

---

## 总结

本文简要介绍了Spring Framework中的事务管理。它是如何创建出和具体底层环境无关的抽象层，以及如何使用声明式/编程式来完成事务管理的。介绍的比较简要，资料参考自[Spring Framework官方文档](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#transaction)。如需要具体的配置方法和更多的例子，可以直接参考上述文档。

