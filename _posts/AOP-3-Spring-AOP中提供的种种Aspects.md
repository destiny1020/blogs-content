---
title: '[AOP] 3. Spring AOP中提供的种种Aspects - Tracing相关'
date: 2017-03-08 22:25:00
categories: ['工具, 库与框架', AOP]
tags: [AOP, Advice, Pointcut, 'Spring AOP', Aspect]
---

在第一篇文章中，介绍了AOP的一些背景知识以及如何快速上手，然后在第二篇中详细分析了AOP的两种实现 - Spring AOP以及AspectJ。

本文偏向于实践，继续介绍Spring AOP中提供的种种Legacy Aspects。虽然这些Aspects的历史已经比较久远了(好多都是在Spring 1.x时代就存在了)，但是并不妨碍我们理解它们背后蕴含的思想以及见识AOP能够解决的问题域。了解这些现成的Aspects，将它们放入我们的工具包，在需要的时候拿出来就能用，而无需重复造轮子；或者将它们有针对性地进行改进，更好地契合我们要解决的问题。

主要会介绍下面几种Aspects，它们都位于包`org.springframework.aop.interceptor`下：

![](http://o6rdpbay0.bkt.clouddn.com/blog/aop/aop-11.jpg)

1. DebugInterceptor
2. CustomizableTraceInterceptor
3. PerformanceMonitorInterceptor
4. AsyncExecutionInterceptor(下一篇文章中介绍)
5. ConcurrencyThrottleInterceptor(下一篇文章中介绍)

<!-- More -->

## Spring AOP(Legacy)基本架构

在介绍具体的Aspect实现之前，先来看看Spring AOP(Legacy)的基本架构：

![](http://o6rdpbay0.bkt.clouddn.com/blog/aop/aop-12.jpg)

Advice是所有Aspect都会实现的一个接口，它实际上是一个Marker Interface：

```java
/**
 * Tag interface for Advice. Implementations can be any type
 * of advice, such as Interceptors.
 *
 * @author Rod Johnson
 * @version $Id: Advice.java,v 1.1 2004/03/19 17:02:16 johnsonr Exp $
 */
public interface Advice {

}
```

这个接口已经是个很老的接口了，从Spring 1.1就开始存在了。Spring作者亲自操刀的代码哦，其实本文介绍的这些Aspects以及基础类型和接口基本上都有他的身影。

然后在Advice下面，有四个子接口：

- Interceptor
- BeforeAdvice
- AfterAdvice
- DynamicIntroductionAdvice

为什么要在本小节的标题里面加上一个(Legacy)呢？主要原因在于本文主要会聚焦于第一个Interceptor。它是Spring在未融合AspectJ时，独立创建出来的一套AOP的解决方案，只不过后来应该是作者发现了AspectJ和Spring AOP有不少业务上有重合之处，因此慢慢走在了一起。因此，基于Interceptor的这套方案不妨称它为Legacy Spring AOP。

关于Interceptor这个接口和由它派生出来的类型，大概是这个样子的(蓝色部分是本文会介绍的几个Aspects)：

![](http://o6rdpbay0.bkt.clouddn.com/blog/aop/aop-13.jpg)

直接继承Advice接口的Interceptor长这样：

```java
/**
 * This interface represents a generic interceptor.
 *
 * // ......
 * This interface is not used directly. Use the sub-interfaces
 * to intercept specific events.
 * // ......
 *
 * @author Rod Johnson
 * @see Joinpoint
 */
public interface Interceptor extends Advice {

}
```

这个Interceptor仍然是个Marker Interface，没有任何实质定义。事实上，Spring一直都在下一盘大棋，所以表现在代码结构上就是预留的扩展点非常多，回过头来看可能结构上有一些臃肿，不够简练。看看2004年和如今2017年，Spring项目的规模差距就知道了。所以从这个意义上而言，学习Spring，主要学习它是如何设计的，如何抽象的。毕竟Spring已经是事实上的JaveEE参考实现了，现如今正统的EJB反倒是无人问津。

这个接口，有2个子接口：

- ConstructorInterceptor
- MethodInterceptor

第一个接口实际上没有任何实现，这里就不讨论它了。
需要讨论的是MethodInterceptor这个接口：

```java
public interface MethodInterceptor extends Interceptor {
	
	/**
	 * Implement this method to perform extra treatments before and
	 * after the invocation. Polite implementations would certainly
	 * like to invoke {@link Joinpoint#proceed()}.
	 * @param invocation the method invocation joinpoint
	 * @return the result of the call to {@link Joinpoint#proceed()};
	 * might be intercepted by the interceptor
	 * @throws Throwable if the interceptors or the target object
	 * throws an exception
	 */
	Object invoke(MethodInvocation invocation) throws Throwable;

}
```

这个接口中定义了一个invoke方法，从注释来看，它的作用和AroundAdvice很相似，会在真正调用目标方法前后执行额外的Before/After操作。而调用`Joinpoint#proceed()`就会去调用目标方法，这个Joinpoint和AspectJ中的JoinPoint概念如出一辙(到底是谁参考的谁暂且不讨论，最后Spring AOP也是基本上投入到了AspectJ的怀抱中)。该方法接受一个MethodInvocation接口实现作为参数，这个接口的层次结构关系如下所示：

![](http://o6rdpbay0.bkt.clouddn.com/blog/aop/aop-14.jpg)

在这个继承链中，定义了如下几个方法：

```java
public interface Joinpoint {

	/**
	 * Proceed to the next interceptor in the chain.
	 * <p>The implementation and the semantics of this method depends
	 * on the actual joinpoint type (see the children interfaces).
	 * @return see the children interfaces' proceed definition
	 * @throws Throwable if the joinpoint throws an exception
	 */
	Object proceed() throws Throwable;

	/**
	 * Return the object that holds the current joinpoint's static part.
	 * <p>For instance, the target object for an invocation.
	 * @return the object (can be null if the accessible object is static)
	 */
	Object getThis();

	/**
	 * Return the static part of this joinpoint.
	 * <p>The static part is an accessible object on which a chain of
	 * interceptors are installed.
	 */
	AccessibleObject getStaticPart();

}

public interface Invocation extends Joinpoint {

	/**
	 * Get the arguments as an array object.
	 * It is possible to change element values within this
	 * array to change the arguments.
	 * @return the argument of the invocation
	 */
	Object[] getArguments();

}

public interface MethodInvocation extends Invocation {

	/**
	 * Get the method being called.
	 * <p>This method is a frienly implementation of the
	 * {@link Joinpoint#getStaticPart()} method (same result).
	 * @return the method being called
	 */
	Method getMethod();

}
```

至此，就定义清楚了完成AOP功能的几个基本要素：

- Interceptor(就是Aspect概念的早期形态)
- Joinpoint(和AspectJ中的JoinPoint表达的是一个意思)

当然，还差一个Pointcut的概念目前没有体现出来。实际上，在早期的Spring AOP中确实没有考虑到这个概念。因为早期的Interceptor都是靠类似下面这种代码被应用的：

```java
public void applyAopProgrammatically() {  
  // 创建代理
  ProxyFactoryBean fb = new ProxyFactoryBean();
  fb.setTarget(new BeanImplementedInterface());
  fb.addAdvice(new DebugInterceptor());
  fb.setBeanFactory(new DefaultListableBeanFactory());
    
  // 获取代理对象
  final IInterface proxy = (IInterface) fb.getObject();
    
  // 基于接口的的代理会采用JDK Dynamic Proxy
  assertTrue(AopUtils.isJdkDynamicProxy(proxy));
  assertFalse(AopUtils.isCglibProxy(proxy));
    
  // 假设IInterface中定义了一个doBiz()方法
  proxy.doBiz();
}
```

以代码的形式应用AOP，在大规模使用的时候是有障碍的，就相当于你需要修改100条记录，我们一般的做法是使用where子句来匹配到这100条记录然后统一修改，而不会是按照ID依次单独去修改。而用AOP的术语表达，where就变成了Pointcut这个概念，它用来表述在什么地方需要应用。所以即便是Legacy Aspects，也可以这样使用：

```xml
<bean id="debugInterceptor" class="org.springframework.aop.interceptor.DebugInterceptor" />

<aop:config>
	<aop:advisor advice-ref="debugInterceptor" pointcut="com.rxjiang.aop.Pointcuts.biz1() || com.rxjiang.aop.Pointcuts.biz2()" />
</aop:config>
```

然后在Java代码中定义Pointcuts：

```java
public class Pointcuts {

  @Pointcut("execution(* (@com.rxjiang.aop.annotation.Biz1 *).*(..))")
  public void biz1() {}

  @Pointcut("execution(* (@com.rxjiang.aop.annotation.Biz2 *).*(..))")
  public void biz2() {}

}
```

当然，现在什么年代了，能用JavaConfig的话就不用XML：

```java
@Configuration
@EnableAspectJAutoProxy
public class AopConfiguration {

  @Bean
  public DebugInterceptor debugInterceptor() {
    return new DebugInterceptor();
  }

  @Bean
  public Advisor debugAdvisor() {
    AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
    pointcut.setExpression("execution(* (@com.rxjiang.aop.annotation.Biz1 *).*(..)) || execution(* (@com.rxjiang.aop.annotation.Biz2 *).*(..))");

    return new DefaultPointcutAdvisor(pointcut, debugInterceptor());
  }

}
```

因此，对于任何被@Biz1或者@Biz2注解标注的方法，就会应用DebugInterceptor它所表达的Aspect。

Legacy Spring AOP的基本架构就介绍到这里，下面开始重点分析几个Interceptor(Aspect)。

## DebugInterceptor

先从简单的开始：

```java
public class DebugInterceptor extends SimpleTraceInterceptor {

  private volatile long count;
  
  // 省略其它代码
  
  @Override
  public Object invoke(MethodInvocation invocation) throws Throwable {
    synchronized (this) {
      this.count++;
    }
    return super.invoke(invocation);
  }

  @Override
  protected String getInvocationDescription(MethodInvocation invocation) {
    return invocation + "; count=" + this.count;
  }
  
  // 省略其它代码
}
```

在它的父类SimpleTraceInterceptor的invokeUnderTrace方法中，旗帜鲜明地表达出它实际上是一个AroundAdvice(一个Entering，一个Exiting)：

```java
@Override
protected Object invokeUnderTrace(MethodInvocation invocation, Log logger) throws Throwable {
  String invocationDescription = getInvocationDescription(invocation);
  logger.trace("Entering " + invocationDescription);
  try {
    Object rval = invocation.proceed();
    logger.trace("Exiting " + invocationDescription);
    return rval;
  } catch (Throwable ex) {
    logger.trace("Exception thrown in " + invocationDescription, ex);
    throw ex;
  }
}
```

而DebugInterceptor的作用也很明显，内部维护了一个线程安全的计数器，记录下来该Interceptor被调用的次数(感觉并没有什么卵用呢)。然后每次执行目标方法的时候都会打印出这个信息。说实话，这个Interceptor的应用场景是十分有限的，毕竟提供的功能太少。而且比较糟糕的一点是，这个地方将日志的接口绑定到了Apache Commons Logging中的一个接口，而现在一般会使用SLF Logging的接口来适配各种可能会使用到的日志实现(比如我个人更喜欢的logback)。

## CustomizableTraceInterceptor

这个Interceptor可以说是DebugInterceptor的扩展。顾名思义它支持定制化的输出方案，能够输出的内容有以下几个：

- 方法名
- 目标方法所在对象类名(完全限定名，短名)
- 实际返回值
- 参数类型列表
- 参数列表
- 异常信息
- 执行时间

在对方法程序进行调试以及性能调优的时候，观察返回值，参数信息以及执行时间还是有一定意义的，比如下面的一段程序就配置了一个主要观察这几类信息：

```java
@Bean
public CustomizableTraceInterceptor customizedInterceptor() {

  CustomizableTraceInterceptor interceptor = new CustomizableTraceInterceptor();
  interceptor.setEnterMessage("Entering $[methodName]($[arguments]).");
  interceptor.setExitMessage(
        "Exiting $[methodName] with return value $[returnValue], took $[invocationTime]ms.");

  return interceptor;
}
```

## PerformanceMonitorInterceptor

其实有了CustomizableTraceInterceptor，这个PerformanceMonitorInterceptor就有些鸡肋了，它实际上就是CustomizableTraceInterceptor的一个子集：记录方法的执行时间。因此这里就不展开了，当然在仅需要记录方法执行时间的时候还是可以考虑直接使用PerformanceMonitorInterceptor的，毕竟记录的东西少一些，额外的开销也会相应地少。

## 总结

本篇文章主要介绍了Spring AOP早期的基本架构以及三个比较实用的用于Tracing的Aspects(早期称为Interceptor，实际上可以将它们视作Around Advice)。

在后续的文章中，会介绍剩下两个和异步并发相关的Interceptors：

- AsyncExecutionInterceptor
- ConcurrencyThrottleInterceptor