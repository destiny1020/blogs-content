---
title: '[AOP] 2. AOP的两种实现-Spring AOP以及AspectJ'
date: 2017-02-26 10:25:00
categories: ['工具, 库与框架', AOP]
tags: [AOP, Advice, Pointcut, 'Spring AOP', AspectJ]
---

在接触Spring以及种类繁多的Java框架时，很多开发人员(至少包括我)都会觉得注解是个很奇妙的存在，为什么加上了@Transactional之后，方法会在一个事务的上下文中被执行呢？为什么加上了@Cacheable之后，方法的返回值会被记录到缓存中，从而让下次的重复调用能够直接利用缓存的结果呢？

随着对AOP的逐渐应用和了解，才明白注解只是一个表象，在幕后Spring AOP/AspectJ做了大量的工作才得以实现这些神奇的功能。

那么，本文就来聊一聊Spring AOP和AspectJ的那些事，它们究竟有什么魔力才让这一切成为现实。

<!-- More -->

## Spring AOP

### 基于代理(Proxy)的AOP实现

首先，这是一种基于代理(Proxy)的实现方式。下面这张图很好地表达了这层关系：

![](http://o6rdpbay0.bkt.clouddn.com/blog/aop/aop-4.jpg)

这张图反映了参与到AOP过程中的几个关键组件(以@Before Advice为例)：

1. 调用者Beans - 即调用发起者，它只知道目标方法所在Bean，并不清楚代理以及Advice的存在
2. 目标方法所在Bean - 被调用的目标方法
3. 生成的代理 - 由Spring AOP为目标方法所在Bean生成的一个代理对象
4. Advice - 切面的执行逻辑

它们之间的调用先后次序反映在上图的序号中：

1. 调用者Bean尝试调用目标方法，但是被生成的代理截了胡
2. 代理根据Advice的种类(本例中是@Before Advice)，对Advice首先进行调用
3. 代理调用目标方法
4. 返回调用结果给调用者Bean(由代理返回，没有体现在图中)

为了理解清楚这张图的意思和代理在中间扮演的角色，不妨看看下面的代码：

```java
@Component
public class SampleBean {
  
  public void advicedMethod() {

  }
  
  public void invokeAdvicedMethod() {
    advicedMethod();
  }

}

@Aspect
@Component
public class SampleAspect {

  @Before("execution(void advicedMethod())")
  public void logException() {
    System.out.println("Aspect被调用了");
  }

}

sampleBean.invokeAdvicedMethod(); // 会打印出 "Aspect被调用了" 吗？
```

`SampleBean`扮演的就是目标方法所在Bean的角色，而`SampleAspect`扮演的则是Advice的角色。很显然，被AOP修饰过的方法是`advicedMethod()`，而非`invokeAdvicedMethod()`。然而，`invokeAdvicedMethod()`方法在内部调用了`advicedMethod()`。那么会打印出来Advice中的输出吗？

答案是**不会**。

如果想不通为什么会这样，不妨再去仔细看看上面的示意图。

这是在使用Spring AOP的时候可能会遇到的一个问题。类似这种间接调用不会触发Advice的原因在于调用发生在目标方法所在Bean的内部，和外面的代理对象可是没有半毛钱的关系哦。我们可以把这个代理想象成一个中介，只有它知道Advice的存在，调用者Bean和目标方法所在Bean知道彼此的存在，但是对于代理或者是Advice却是一无所知的。因此，没有通过代理的调用是绝无可能触发Advice的逻辑的。如下图所示：

![](http://o6rdpbay0.bkt.clouddn.com/blog/aop/aop-5.jpg)

### Spring AOP的两种实现方式

Spring AOP有两种实现方式：

- 基于接口的动态代理(Dynamic Proxy)
- 基于子类化的CGLIB代理

我们在使用Spring AOP的时候，一般是不需要选择具体的实现方式的。Spring AOP能根据上下文环境帮助我们选择一种合适的。那么是不是每次都能够这么"智能"地选择出来呢？也不尽然，下面的例子就反映了这个问题：

```java
@Component
public class SampleBean implements SampleInterface {

  public void advicedMethod() {

  }

  public void invokeAdvicedMethod() {
    advicedMethod();
  }

}

public interface SampleInterface {}
```

在上述代码中，我们为原来的Bean实现了一个新的接口`SampleInterface`，这个接口中并没有定义任何方法。这个时候，再次运行相关测试代码的时候就会出现异常(摘录了部分异常信息)：

```
org.springframework.beans.factory.BeanCreationException: 
Error creating bean with name 'com.destiny1020.SampleBeanTest': 
Injection of autowired dependencies failedCaused by: 
org.springframework.beans.factory.NoSuchBeanDefinitionException: 
No qualifying bean of type [com.destiny1020.SampleBean] found for dependency: 
expected at least 1 bean which qualifies as autowire candidate for this dependency. 
```

也就是说在Test类中对于Bean的Autowiring失败了，原因是创建SampleBeanTest Bean的时候发生了异常。那么为什么会出现创建Bean的异常呢？从异常信息来看并不明显，实际上这个问题的根源在于Spring AOP在创建代理的时候出现了问题。

这个问题的根源可以在这里得到一些线索：

[Spring AOP Reference - AOP Proxies](https://docs.spring.io/spring/docs/current/spring-framework-reference/html/aop.html#aop-introduction-proxies)

文档中是这样描述的(每段后加上了翻译)：

> Spring AOP defaults to using standard JDK dynamic proxies for AOP proxies. This enables any interface (or set of interfaces) to be proxied.
> 
> Spring AOP默认使用标准的JDK动态代理来实现AOP代理。这能使任何借口(或者一组接口)被代理。
> 
> Spring AOP can also use CGLIB proxies. This is necessary to proxy classes rather than interfaces. CGLIB is used by default if a business object does not implement an interface. As it is good practice to program to interfaces rather than classes; business classes normally will implement one or more business interfaces. It is possible to force the use of CGLIB, in those (hopefully rare) cases where you need to advise a method that is not declared on an interface, or where you need to pass a proxied object to a method as a concrete type.
> 
> Spring AOP也使用CGLIB代理。对于代理classes而非接口这是必要的。如果一个业务对象没有实现任何接口，那么默认会使用CGLIB。由于面向接口而非面向classes编程是一个良好的实践；业务对象通常都会实现一个或者多个业务接口。强制使用CGLIB也是可能的(希望这种情况很少)，此时你需要advise的方法没有被定义在接口中，或者你需要向方法中传入一个具体的对象作为代理对象。

因此，上面异常的原因在于：
> 强制使用CGLIB也是可能的(希望这种情况很少)，此时你需要advise的方法没有被定义在接口中。

我们需要advise的方法是SampleBean中的advicedMethod方法。而在添加接口后，这个方法并没有被定义在该接口中。所以正如文档所言，我们需要强制使用CGLIB来避免这个问题。

强制使用CGLIB很简单：

```java
@Configuration
@EnableAspectJAutoProxy(proxyTargetClass = true)
@ComponentScan(basePackages = "com.destiny1020")
public class CommonConfiguration {}
```

向`@EnableAspectJAutoProxy`注解中添加属性`proxyTargetClass = true`即可。
CGLIB实现AOP代理的原理是通过动态地创建一个目标Bean的子类来实现的，该子类的实例就是AOP代理，它建立起了目标Bean到Advice的联系。

当然还有另外一种解决方案，那就是将方法定义声明在新创建的接口中并且去掉之前添加的`proxyTargetClass = true`：

```java
@Component
public class SampleBean implements SampleInterface {

  @Override
  public void advicedMethod() {

  }

  @Override
  public void invokeAdvicedMethod() {
    advicedMethod();
  }

}

public interface SampleInterface {

  void invokeAdvicedMethod();

  void advicedMethod();

}

@Configuration
@EnableAspectJAutoProxy
@ComponentScan(basePackages = "com.destiny1020")
public class CommonConfiguration {}
```

这样就让业务对象实现了一个接口，从而能够使用基于标准JDK的动态代理来完成Spring AOP代理对象的创建。

从Debug Stacktrace的角度也可以看出这两种AOP实现方式上的区别：

- JDK动态代理

![](http://o6rdpbay0.bkt.clouddn.com/blog/aop/aop-7.jpg)

- CGLIB

![](http://o6rdpbay0.bkt.clouddn.com/blog/aop/aop-8.jpg)

关于动态代理和CGLIB这两种方式的简要总结如下：

- JDK动态代理(Dynamic Proxy)
	- 基于标准JDK的动态代理功能
	- 只针对实现了接口的业务对象

- CGLIB
	- 通过动态地对目标对象进行子类化来实现AOP代理，上面截图中的`SampleBean$$EnhancerByCGLIB$$1767dd4b`即为动态创建的一个子类
	- 需要指定`@EnableAspectJAutoProxy(proxyTargetClass = true)`来强制使用
	- 当业务对象没有实现任何接口的时候默认会选择CGLIB

---

## AspectJ

AspectJ是Eclipse旗下的一个项目。至于它和Spring AOP的关系，不妨可将Spring AOP看成是Spring这个庞大的集成框架为了集成AspectJ而出现的一个模块。

毕竟很多地方都是直接用到AspectJ里面的代码。典型的比如`@Aspect`，`@Around`，`@Pointcut`注解等等。而且从相关概念以及语法结构上而言，两者其实非常非常相似。比如Pointcut的表达式语法以及Advice的种类，都是一样一样的。

那么，它们的区别在哪里呢？

最大的区别在于两者实现AOP的底层原理不太一样：

- Spring AOP: 基于代理(Proxying)
- AspectJ: 基于字节码操作(Bytecode Manipulation)

用一张图来表示AspectJ使用的字节码操作，就一目了然了：

![](http://o6rdpbay0.bkt.clouddn.com/blog/aop/aop-6.jpg)

通过编织阶段(Weaving Phase)，对目标Java类型的字节码进行操作，将需要的Advice逻辑给编织进去，形成新的字节码。毕竟JVM执行的都是Java源代码编译后得到的字节码，所以AspectJ相当于在这个过程中做了一点手脚，让Advice能够参与进来。

而编织阶段可以有两个选择，分别是加载时编织(也可以成为运行时编织)和编译时编织：

### 加载时编织(Load-Time Weaving)

顾名思义，这种编织方式是在JVM加载类的时候完成的。

使用它需要进行相关的配置，举例如下：

在类路径的META-INF目录下创建一个文件名为aop.xml:

```xml
<aspectj>
  <weaver>
    <include within="com.destiny1020..*" />
  </weaver>
  <aspects>
    <aspect name="com.destiny1020.SampleAspect" />
  </aspects>
</aspectj>
```

然后添加启动参数，直接使用AspectJ提供的或者使用Spring提供的工具：

```
# AspectJ
-javaagent:path_to/aspectjweaver-{version}.jar

# Spring
-javaagent:path_to/org.springframework.instrument-{version}.jar
```

当使用Spring提供的工具时，还需要进行一些配置，以JavaConfig为例：

```java
@Configuration
@EnableLoadTimeWeaving
@ComponentScan(basePackages = "com.destiny1020")
public class CommonConfiguration {}
```

重点就是上述的`@EnableLoadTimeWeaving`。

### 编译时编织(Compile-Time Weaving)

需要使用AspectJ的编译器来替换JDK的编译器。可以借助Maven AspectJ来实现，下面是一例：

```
<plugin>
	<groupId>org.codehaus.mojo</groupId>
	<artifactId>aspectj-maven-plugin</artifactId>
	<version>1.4</version>
	<dependencies>
		<dependency>
			<groupId>org.aspectj</groupId>
			<artifactId>aspectjrt</artifactId>
			<version>${aspectj.version}</version>
		</dependency>
		<dependency>
			<groupId>org.aspectj</groupId>
			<artifactId>aspectjtools</artifactId>
			<version>${aspectj.version}</version>
		</dependency>
	</dependencies>
	<executions>
		<execution>
			<phase>process-sources</phase>
			<goals>
				<goal>compile</goal>
				<goal>test-compile</goal>
			</goals>
		</execution>
	</executions>
	<configuration>
		<outxml>true</outxml>
		<source>${java.version}</source>
		<target>${java.version}</target>
	</configuration>
</plugin>
```

然后直接通过`mvn test`进行测试：

![](http://o6rdpbay0.bkt.clouddn.com/blog/aop/aop-9.jpg)

**自定义的编译错误/警告**

举个例子，有两个Service1和Service2分别位于两个包Package1和Package2下，只能在Package2中调用来自本包内部的方法，在Service1中调用Service2中提供的方法会导致编译错误(能够用访问控制符解决的问题强行用这种方式来解决，当然只是为了说明问题：）)：

```java
@Aspect
public class EmitCompilationErrorAspect {

  @DeclareError("call (* com.destiny1020.biz.package2..*.*(..))"
      + "&& !within(com.destiny1020.biz.package2..*)")
  public static final String notInBizPackage2 = "只能在Package2中调用来自Package2的方法";

}
```

```java
package com.destiny1020.biz.package1;

import com.destiny1020.biz.package2.ServiceInPackage2;

public class ServiceInPackage1 {

  ServiceInPackage2 service2 = new ServiceInPackage2();

  public void invokeMethodInPackage2() {
    service2.doBizInPackage2();  // 这里理应会出现编译错误
  }

}
```

实际情况也正式如此：

![](http://o6rdpbay0.bkt.clouddn.com/blog/aop/aop-10.jpg)

在声明编译错误Pointcut的时候，出现了两个新概念：

- call
- within

这两个新出现的Pointcut原语只能在使用AspectJ作为AOP实现的时候才可用。它们表达的是什么意思呢：

- call：针对所有的调用者(caller)，即哪里调用了Pointcut表达式匹配的方法，在该方法被执行之前就会被匹配到；而我们经常使用的execution则是针对所有的被调用方法，而不会care是谁调用的该方法
- within：这个很好理解，它的Pointcut表达式是一个用来匹配完整限定类名的表达式，比如上例中的`!within(com.destiny1020.biz.package2..*)`意味不在包`com.destiny1020.biz.package2`中的类。

在使用AspectJ的编译时编织功能时，由于使用了AspectJ Compiler来完成代码的编译，因此可以根据编码规范添加相应的编译错误/警告，来进一步地让代码更加规范。这个特性对于辅助实现大型项目的编码规范还是很有益处的。

## 哪种方式更好

先下结论：It Depends.

得根据具体需求，不过我个人认为在对AOP的需求不那么深入和迫切的时候，使用Spring AOP足矣。

毕竟Spring作为一个以集成起家的框架，在设计Spring AOP的时候也是为了减轻开发人员负担而做了不少努力的。它提供的开箱即用(Out-of-the-box)的众多AOP功能让很多开发人员甚至都不知道什么是AOP，就算知道了AOP是Spring的一大基石或者@Transactional和@Cacheable等等常用注解是借助了AOP的力量，但是再深入恐怕就有点勉为其难了。这是优点也是缺点，当需要对AOP的实现做出精细化调整的时候，就会有力不从心的感觉。

这个时候，就可以考虑使用AspectJ。AspectJ的功能更加全面和强大。支持全部的Pointcut类型。

[这里](https://stackoverflow.com/questions/1606559/spring-aop-vs-aspectj)进行了一个简单的比较，摘录并简单翻译(括号内是我添加的补充)如下：

**Spring-AOP Pros**

- 比AspectJ更简单，不需要使用Load-Time Weaving以及AspectJ编译器(为了Compile-Time Weaving)
- 当使用@Aspect注解时可以很方便的迁移到AspectJ AOP实现
- 使用代理模式和装饰模式

**Spring-AOP Cons**

- 由于是基于代理的AOP，所以基本上只能选择方法execution这一个Pointcut原语
- 在类本身中调用另一个方法的时候Aspects不会生效
- 有一点运行时的额外开销
- 无法为不是从Spring Factory中创建的对象添加Aspect(只对Spring Bean有效)

---

**AspectJ Pros**

- 支持所有的Pointcut原语，这意味着你可以做任何事情
- 运行时开销比Spring AOP少
- 能够添加各种编译错误来保障代码质量(这一条是我自己添加的)

**AspectJ Cons**

- 当心。检查是否发生了意料之外的Weaving操作
- 使用Compile-Time Weaving时需要额外的构建步骤(使用AspectJ Compiler)，在使用Load-Time Weaving时需要一些配置(-javaassist)

## 参考资料

[Spring AOP Reference](https://docs.spring.io/spring/docs/current/spring-framework-reference/html/aop.html)

[AspectJ Quick Reference](https://eclipse.org/aspectj/doc/next/quick5.pdf)








