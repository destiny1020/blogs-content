---
title: '[AOP] 1. AOP的由来以及快速上手'
date: 2017-02-25 14:44:00
categories: ['工具, 库与框架', AOP]
tags: [AOP, Advice, Pointcut, 'Spring AOP', AspectJ]
---

## AOP从何而来

技术的演化从来都不是随机现象。往往都是为了应对某种特定的问题，而形成的一系列切实可行解决方案或者优雅的最佳实践，然后把它们汇聚在一起，就形成了一个工具，一个库或者是一个框架。

### 为应对Cross-cutting问题而生

要了解AOP(Aspect Oriented Programming，面向切面编程)从何而来，首先来看看下面这段代码：

```java
public void doBusinessLogic() {
  logger.trace("进入 " + CLASS_NAME + "." + METHOD_NAME);
  TransactionStatus tx = transactionManager.getTransaction(new DefaultTransactionDefinition()); 
  try {
    // 开始执行业务逻辑
    // ......
    // 业务逻辑结束
  } catch (Exception e) {
    logger.error("异常 " + CLASS_NAME + "." + METHOD_NAME, e); 
    tx.setRollbackOnly();
    throw e;
  } finally { 
    transactionManager.commit(tx);
    logger.trace("退出 " + CLASS_NAME + "." + METHOD_NAME);
  } 
}
```

<!-- More -->

发现上面这段代码有什么问题了吗？很明显，样板代码(Boilerplate)太多了，真正重要的业务逻辑反而只占了很小的一部分(如果执行的业务逻辑比较简单的话)。这种代码结构合理吗？显然不合理。上述的样板代码还只包含了简单的Tracing，Transaction以及Exception处理，如果还需要更多的这类处理，那么代码将臃肿不堪。

所以，为了处理这类公用需求，最大程度地避免重复代码而遵循DRY原则。AOP应运而生。那么AOP要解决一个什么问题呢？

![](http://o6rdpbay0.bkt.clouddn.com/blog/aop/aop-1.jpg)

上面这张图能够说明要解决的问题。

每种颜色的代码就相当于公用的需求点，比如绿色的Tracing，蓝色的Transaction以及红色的Exception Handling。这些代码存在于不止一个类中，通常而言是分散的到处都是，就像上面的那段代码一样。

而这些重复而通用的代码就是AOP要解决的问题，每一个共同的横切关注点(Cross-cutting Concern)对应于一个切面(Aspect)。这些切面的目的就是将四处散落的通用代码集中管理，然后采用声明式的方式将这些代码再注入到需要执行的位置。

### AOP概述

Aspect， Advice以及Pointcut的关系。

那么，什么是切面(Aspect)呢？

回顾一下上面的那张示意图。里面表达了切面的两个要素：

1. 执行什么，比如Tracing亦或是Transaction
2. 在哪执行，比如是在Class A还是Class B

那么反映到AOP的概念中，这两个要素分别对应的是Advice以及Pointcut。

所以，简而言之：Aspect = Advice(做什么) + Pointcut(在哪做)

比如对于上面的Tracing功能而言：

1. Advice - 执行Logging相关操作
2. Pointcut - 在方法的开始和结束处执行

## 快速上手

### 一个简单的用于Tracing的例子

```java
@Component
@Aspect
public class TracingAspect {
  private Logger logger = LoggerFactory.getLogger(TracingAspect.class);

  @Before("execution(* *(..))")
  public void beforeLogging(JoinPoint joinPoint) {
    logger.trace("进入 " + joinPoint.getStaticPart().getSignature().toString());
  }
}
```

这段代码实现了一个在执行方法时使用日志记录Tracing信息的切面(Aspect)。它的逻辑很清晰：

1. 使用@Aspect注解说明这是一个切面(Aspect)
2. 使用@Component注解说明这是一个Spring Bean
3. 使用@Before注解来表达切面的类型，也就是在方法执行之前会首先调用Advice
4. 在@Before注解中使用表达式`execution(* *(..))`来表达Pointcut的概念
5. 实现beforeLogging方法，它代表了具体的Advice
6. 使用JoinPoint参数来得到将要调用的方法信息

下面我们来看看如何在工程中启用AOP。

### 如何工程中启用

在一个传统的Spring项目中，可以采用XML或者Java Config的方式来开启对于AOP的支持：

- XML的配置方式

```xml
<beans>
  <aop:aspectj-autoproxy />
  <context:component-scan base-package="com.destiny1020" />
</beans>
```

`<aop:aspectj-autoproxy />`的功能是开启对于@Aspect的支持。

- Java Config的配置方式

```java
@Configuration
@EnableAspectJAutoProxy 
@ComponentScan(basePackages="com.destiny1020") 
public class AOPConfiguration {}
```

上述的`@EnableAspectJAutoProxy`就对应着XML配置中的`<aop:aspectj-autoproxy />`

## Advice的5种类型

Advice的类型有5种：

![](http://o6rdpbay0.bkt.clouddn.com/blog/aop/aop-2.jpg)

简单介绍如下：

### Before Advice

上面的例子中使用的就是Before Advice。它使用@Before注解来表达。

对于这类Advice，有几个注意事项：

1. Advice的执行在执行目标方法之前(并没有进入目标方法)
2. 如果Advice的执行中抛出了异常，那么会阻止目标方法运行，异常会被传递给目标方法的调用者

### After Advice

一个简单的例子：

```java
@After("execution(* *(..))")
public void afterLogging(JoinPoint joinPoint) {
  logger.trace("退出 " + joinPoint.getSignature());
	
  // 获取调用目标方法时的参数
  for (Object arg : joinPoint.getArgs()) {
    logger.trace("参数 : " + arg);
  }
}
```

关于Pointcut表达式`execution(* *(..))`会在后面Pointcut一节中进行介绍。

对于After Advice，有几个注意事项：

1. Advice的执行在执行目标方法之后(退出目标方法后)
2. 如果目标方法抛出了异常，After Advice仍然会被执行

### Around Advice

Around Advice是功能最强大的一种Advice。

下面是一例：

```java
@Around("execution(* *(..))")
public Object aroundAdvice(ProceedingJoinPoint pjp) throws Throwable {
  String minfo = pjp.getStaticPart().getSignature().toString();
  logger.trace("进入 " + minfo);
  try {
    return pjp.proceed();
  } catch (Throwable ex) {
    logger.error("异常 " + minfo, ex);
    throw ex;
  } finally {
    logger.trace("退出 " + minfo);
  }
}
```

如上面的图片所示，Around Advice可以被看成目标方法的一个Wrapper。它能够灵活地控制何时调用甚至不调用原本的目标方法。它有以下几个特点和注意事项：

1. 需要接受一个`ProceedingJoinPoint`作为参数，通过调用它的proceed方法来调用目标方法
2. 能够捕获目标方法可能抛出的异常，所有Advice种类中只有Around Advice能够做到
3. 能够修改目标方法的返回结果，所有Advice种类中只有Around Advice能够做到
4. 能够直接忽略目标方法，所有Advice种类中只有Around Advice能够做到

因此，Around Advice是所有Advice中功能最强大的一个。更灵活与更强大也就意味着更大的责任，它的使用也确实也相对复杂一些。在应用它的时候需要仔细调试确保能够满足所有的业务需求且不产生副作用。

### After Returning Advice

一个例子：

```java
 
@AfterReturning(pointcut = "execution(* *(..))", returning = "result")
public void returnLogging(String result) { 
  logger.trace("结果 "+ result);
}
```

值得注意的地方有：

1. 只有在目标方法成功执行完毕后才会执行Advice
2. 可以通过指定returning属性指定目标方法的返回类型，即只有特定类型的结果被返回的时候才会触发Advice的执行。returning指定的值和参数的名称需要相同，比如上述的result

### After Throwing Advice

一个例子：

```java
@AfterThrowing(pointcut = "execution(* *(..))", throwing = "iae")
public void throwLogging(IllegalArgumentException iae) {
  logger.error("非法参数异常 ", iae);
}
```

值得注意的地方有：

1. 只有在目标方法抛出异常时才会执行Advice
2. 抛出的异常类型可以作为参数传入到Advice实现方法中，异常最终也会被抛出给调用者
3. 可以通过指定throwing属性指定抛出的类型异常，即只有特定类型的异常被抛出的时候才会触发Advice的执行。throwing指定的值和参数的名称需要相同，比如上述的iae

## Pointcut的声明和使用方式

### Pointcut语法规则

拿上面一直出现的`execution(* *(..))`作为例子：

![](http://o6rdpbay0.bkt.clouddn.com/blog/aop/aop-3.jpg)

所以，Pointcut有几个重要的组成部分：

1. Pointcut原语(Primitive Pointcut) - 表明Advice介入的阶段，上述的execution表明介入的阶段是在目标方法的执行期间，除此之外还有很多的别的介入时机，具体可以参考文末的相关资料。
2. Pointcut表达式 - 每种原语对应着自己的表达式语法规则，就拿最常见的execution而言，它的表达式形如`* *(..)`
	1. 第一个* ：任意的返回类型
	2. 第二个* ：任意的方法名称(可以为带上包名和类名的完整限定名)
	3. (..) ：任意的参数类型，不限定个数，不限定类型

举几个例子：

- `execution(* register())` - 匹配所有不接受参数，名为register的方法，不限返回类型
- `execution(int register(int, int))` - 匹配接受两个int类型作为参数，名为register的方法，且返回类型为int类型
- `execution(* register(*))` - 匹配接受1个不限定参数类型，名为register的方法，不限返回类型
- `execution(* com.destiny1020.AuthService.register(..)` - 匹配`com.destiny1020.AuthService`类下的所有register重载
- `execution(* com.destiny1020..*AuthService.register(..)` - 匹配`com.destiny1020`子包下的类名以AuthService结尾的所有类中的register重载

### 使用注解的情况

另外，还可以利用注解来限定Pointcut的范围。这个特性其实我们用得最多，诸如`@Transactional`，`@Cacheable`等等都可以归为这一类。那么如何在Pointcut表达式中进行声明呢，实际上可以分为两种情况：

1. 方法注解 - `execution(@com.destiny1020.anno.LoadBalanced * *(..))`
2. 类注解 - `execution(* (@com.destiny1020.anno.LoadBalanced *).*(..))`

也就是说，一旦方法或者类被指定的注解给标注了，那么该方法或者类中的所有方法都会被定义为Advice的目标方法。需要注意的是，注解的名称需要是带有完整包名的限定名。

### 使用逻辑操作符的情况

在Pointcut表达式中还能够使用逻辑操作符：

比如这个表达式：

`execution(* com.destiny1020.service..*.load(..)) || execution(* com.destiny1020.repository..*.load(..))`

它的意思很直观，即匹配service包或者repository包下的所有名为load的方法及其重载，不考虑参数数量和类型，也不考虑返回类型。

### Pointcut的重用

在实际的工作中可能会出现重复编写Pointcut的情况，与其到处粘贴复制，有没有一种方法能够仅仅定义一次呢，答案是可以通过`@Pointcut`注解来帮我们：

```java
public class PointcutDefinitions {
  @Pointcut("execution(@com.destiny1020.anno.LoadBalanced * *(..))") 
  public void loadBalancedAnnotated() {
    // 空方法就OK
  }
} 
```

然后在Advice的注解中使用即可：

```java
@Around("PointcutDefinitions.loadBalancedAnnotated()")
public void trace(ProceedingJoinPoint pjp) throws Throwable { 
  // 定义Advice的逻辑
}
```

这样就可以将所有的Pointcut集中定义，不会把复杂而难以理解(至少乍一眼看上去是如此)的Pointcut弄的到处都是而难以维护了。

在下一篇文章中，会探讨AOP的两种实现：

- Spring AOP
- AspectJ

## 参考资料

[Spring AOP Reference](https://docs.spring.io/spring/docs/current/spring-framework-reference/html/aop.html)

[AspectJ Quick Reference](https://eclipse.org/aspectj/doc/next/quick5.pdf)









