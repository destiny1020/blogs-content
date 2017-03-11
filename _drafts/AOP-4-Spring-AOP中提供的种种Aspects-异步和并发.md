---
title: '[AOP] 4. Spring AOP中提供的种种Aspects - 异步和并发'
date: 2017-03-09 11:25:00
categories: ['工具, 库与框架', AOP]
tags: [AOP, Advice, Pointcut, 'Spring AOP', Aspect, Concurrency]
---

上一篇文章介绍了Spring AOP中提供的种种与Tracing相关的Aspects，还剩两个Aspects没有讨论：

- AsyncExecutionInterceptor
- ConcurrencyThrottleInterceptor

本文继续探讨和异步与并发相关一个Aspect，也是使用的比较普遍的一个：

- AsyncExecutionInterceptor

在下篇文章中会继续讨论ConcurrencyThrottleInterceptor。

## AsyncExecutionInterceptor

首先来看看这个类的和其相关类的一个层次关系：

![](http://o6rdpbay0.bkt.clouddn.com/blog/aop/aop-15.jpg)

<!-- More -->

从这个类的文档来看：

> AOP Alliance that processes method invocations asynchronously, using a given {@link org.springframework.core.task.AsyncTaskExecutor}. Typically used with the {@link org.springframework.scheduling.annotation.Async} annotation.

它交待了两件事情：

1. 异步执行是通过AsyncTaskExecutor来执行的
2. 通常和@Async这个注解一起使用

因此，就先来看看AsyncTaskExecutor和它的父接口们是如何定义的：

### AsyncTaskExecutor执行者的抽象层次

#### Executor

层次结构上的最顶层是位于并发包内的Executor接口。

```java
public interface Executor {

    /**
     * Executes the given command at some time in the future.  The command
     * may execute in a new thread, in a pooled thread, or in the calling
     * thread, at the discretion of the {@code Executor} implementation.
     *
     * @param command the runnable task
     * @throws RejectedExecutionException if this task cannot be
     * accepted for execution
     * @throws NullPointerException if command is null
     */
    void execute(Runnable command);
}
```

就定义了一个execute方法，接受一个Runnable作为待执行任务的定义。它的目的在于将任务和执行任务的基础设施(新创建的线程/线程池中的线程/当前调用线程)解耦。即任务并不清楚它自身将会被谁给执行。

#### TaskExecutor

位于`org.springframework.core.task`包中的接口。它直接覆盖了Executor接口中对于execute方法的定义。并且将原来的参数名称从command改成了task。其实这也更清晰地表达了Runnable接口的实质意义，它代表的是一些可执行的指令，这些可执行的指令构成了一个任务。

```java
public interface TaskExecutor extends Executor {

	/**
	 * Execute the given {@code task}.
	 * <p>The call might return immediately if the implementation uses
	 * an asynchronous execution strategy, or might block in the case
	 * of synchronous execution.
	 * @param task the {@code Runnable} to execute (never {@code null})
	 * @throws TaskRejectedException if the given task was not accepted
	 */
	@Override
	void execute(Runnable task);

}
```

这里同时提到了这个方法可能会立即返回(当异步执行的时候)，也可能会被阻塞(当同步执行的时候，即有可能使用的是当前调用线程来执行该任务)。

#### AsyncTaskExecutor

```java
public interface AsyncTaskExecutor extends TaskExecutor {

	/** Constant that indicates immediate execution */
	long TIMEOUT_IMMEDIATE = 0;

	/** Constant that indicates no time limit */
	long TIMEOUT_INDEFINITE = Long.MAX_VALUE;


	/**
	 * Execute the given {@code task}.
	 * @throws TaskRejectedException if the given task was not accepted
	 */
	void execute(Runnable task, long startTimeout);

	/**
	 * Submit a Runnable task for execution, receiving a Future representing that task.
	 * @since 3.0
	 */
	Future<?> submit(Runnable task);

	/**
	 * Submit a Callable task for execution, receiving a Future 
	 * @since 3.0
	 */
	<T> Future<T> submit(Callable<T> task);

}
```

这个接口定义了三个方法。一个方法作为execute方法的重载，添加了一个startTimeout参数用于指定待执行任务的紧急程度。接口中定义的两个常量就是描述该信息的。比如使用TIMEOUT_IMMEDIATE就意味着当前任务很紧急，需要立即执行。但是这个参数也只是给底层的任务执行者一个提示(Hint)，是否参考还要看具体实现。而默认不使用startTimeout参数的重载会使用TIMEOUT_INDEFINITE作为默认值，此时任务的优先级定义的是比较低的。

另外两个submit方法则是为异步任务执行添加的，在提交任务后会得到一个Future对象作为获取结果的凭证，这个Future可以理解成JavaScript中的Promise。两者本质上就是同样的概念。同时，还支持了基于Callable接口的任务，Callable和Runnable最大的区别就在于前者有返回值的概念，因此它也更贴近于Task的概念。

### AsyncExecutionInterceptor的层次结构

关于它的父类：

```java
/**
 * Base class for asynchronous method execution aspects, such as
 * {@code org.springframework.scheduling.annotation.AnnotationAsyncExecutionInterceptor}
 * or {@code org.springframework.scheduling.aspectj.AnnotationAsyncExecutionAspect}.
 *
 * <p>Provides support for <i>executor qualification</i> on a method-by-method basis.
 * {@code AsyncExecutionAspectSupport} objects must be constructed with a default {@code
 * Executor}, but each individual method may further qualify a specific {@code Executor}
 * bean to be used when executing it, e.g. through an annotation attribute.
 *
 * @author Chris Beams
 * @author Juergen Hoeller
 * @author Stephane Nicoll
 * @since 3.1.2
 */
public abstract class AsyncExecutionAspectSupport implements BeanFactoryAware {
	// 省略实现部分
}
```

从Javadoc来看的话，这个类提供了异步方法执行Aspects所需的一些基础支持，比如：

- AnnotationAsyncExecutionInterceptor：它实际上是基于Spring AOP的一个实现
- AnnotationAsyncExecutionAspect：它则是基于AspectJ的实现，从包名就可以看出来

完整的实现部分就不贴出来了，谈谈这个类的主要功能：

- 待执行方法和具体Executor的一个映射关系：

```java
private final Map<Method, AsyncTaskExecutor> executors = new ConcurrentHashMap<Method, AsyncTaskExecutor>(16);

protected AsyncTaskExecutor determineAsyncExecutor(Method method) {
  AsyncTaskExecutor executor = this.executors.get(method);
  if (executor == null) {
    Executor targetExecutor;
    String qualifier = getExecutorQualifier(method);
    if (StringUtils.hasLength(qualifier)) {
      targetExecutor = findQualifiedExecutor(this.beanFactory, qualifier);
    } else {
      targetExecutor = this.defaultExecutor;
      if (targetExecutor == null) {
        synchronized (this.executors) {
          if (this.defaultExecutor == null) {
            this.defaultExecutor = getDefaultExecutor(this.beanFactory);
          }
          targetExecutor = this.defaultExecutor;
        }
      }
    }
    if (targetExecutor == null) {
      return null;
    }
    executor = (targetExecutor instanceof AsyncListenableTaskExecutor ?
					(AsyncListenableTaskExecutor) targetExecutor : new TaskExecutorAdapter(targetExecutor));
    this.executors.put(method, executor);
  }
  return executor;
}

protected abstract String getExecutorQualifier(Method method);
```

因此，运行时的一个方法对象(Method)的执行和具体执行它的Executor是一个动态的关联关系。getExecutorQualifier的返回值用来获取到这个Executor实例。如果不提供qualifier的话，会使用一个默认的Executor：

```java
protected Executor getDefaultExecutor(BeanFactory beanFactory) {
  if (beanFactory != null) {
    try {
      // Search for TaskExecutor bean... not plain Executor since that would
      // match with ScheduledExecutorService as well, which is unusable for
      // our purposes here. TaskExecutor is more clearly designed for it.
      return beanFactory.getBean(TaskExecutor.class);
    } catch (NoUniqueBeanDefinitionException ex) {
      try {
        return beanFactory.getBean(DEFAULT_TASK_EXECUTOR_BEAN_NAME, Executor.class);
      } catch (NoSuchBeanDefinitionException ex2) {
        if (logger.isInfoEnabled()) {
          logger
              .info("More than one TaskExecutor bean found within the context, and none is named "
                  + "'taskExecutor'. Mark one of them as primary or name it 'taskExecutor' (possibly "
                  + "as an alias) in order to use it for async processing: "
                  + ex.getBeanNamesFound());
        }
      }
    } catch (NoSuchBeanDefinitionException ex) {
      logger.debug("Could not find default TaskExecutor bean", ex);
      // Giving up -> either using local default executor or none at all...
      logger.info("No TaskExecutor bean found for async processing");
    }
  }
  return null;
}
```

即最终的默认Executor是一个TaskExecutor接口的实现类。做个实验来看看默认的Executor到底是啥(基于Spring 4.3.7)：

```java
@EnableAsync
@ComponentScan(basePackages = "com.rxjiang")
public class SystemConfiguration {
  // ......
}

@Service
public class PlainService
  @Async
  public void doSomethingAsync() {
    System.out.println(Thread.currentThread().getName() + ": Async Biz...");
  }
}

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = {SystemConfiguration.class})
public class AsyncTest {

  @Autowired
  private PlainService service;

  @Test
  public void testAsync() {
    service.doSomethingAsync();
  }

}
```

打印出来的结果：

```
SimpleAsyncTaskExecutor-1: Async Biz...
```

因此最终来执行异步任务的是SimpleAsyncTaskExecutor，具体是在类中完成创建工作的：

```java
@Override
protected Executor getDefaultExecutor(BeanFactory beanFactory) {
  Executor defaultExecutor = super.getDefaultExecutor(beanFactory);
  return (defaultExecutor != null ? defaultExecutor : new SimpleAsyncTaskExecutor());
}
```

尝试了通过父类AsyncExecutionAspectSupport中的getDefaultExecutor来得到Executor，但是未遂(返回了null)，也就是说Spring并没有帮助我们定义一个TaskExecutor接口的实现类作为默认Executor。

#### SimpleAsyncTaskExecutor - 默认异步Executor

来看看SimpleAsyncTaskExecutor是怎么一回事：

```java
/**
 * {@link TaskExecutor} implementation that fires up a new Thread for each task,
 * executing it asynchronously.
 *
 * <p>Supports limiting concurrent threads through the "concurrencyLimit"
 * bean property. By default, the number of concurrent threads is unlimited.
 *
 * <p><b>NOTE: This implementation does not reuse threads!</b> Consider a
 * thread-pooling TaskExecutor implementation instead, in particular for
 * executing a large number of short-lived tasks.
 *
 * @author Juergen Hoeller
 * @since 2.0
 * @see #setConcurrencyLimit
 * @see SyncTaskExecutor
 * @see org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor
 * @see org.springframework.scheduling.commonj.WorkManagerTaskExecutor
 */
@SuppressWarnings("serial")
public class SimpleAsyncTaskExecutor extends CustomizableThreadCreator implements AsyncListenableTaskExecutor, Serializable {
  // ...
}
```

这个简单的异步Executor的策略如文档所言，为了每个任务都会新创建一个线程。通过一个名为concurrencyLimit的属性控制并发度。而且，对于执行大量短任务，不推荐使用它。这种场景下使用基于线程池的Executor更合适。

关于所谓的控制并发度，考虑在下篇文章介绍ConcurrencyThrottleInterceptor的时候一并讨论吧。

他的父类CustomizableThreadCreator功能也很简单，就是设置所创建的线程的各种属性，比如ThreadGroup，Thread Name等等。

值得一提的是，这个Executor还实现了AsyncListenableTaskExecutor接口：

```java
public interface AsyncListenableTaskExecutor extends AsyncTaskExecutor {

	/**
	 * Submit a {@code Runnable} task for execution, receiving a {@code ListenableFuture}
	 * representing that task. The Future will return a {@code null} result upon completion.
	 * @param task the {@code Runnable} to execute (never {@code null})
	 * @return a {@code ListenableFuture} representing pending completion of the task
	 * @throws TaskRejectedException if the given task was not accepted
	 */
	ListenableFuture<?> submitListenable(Runnable task);

	/**
	 * Submit a {@code Callable} task for execution, receiving a {@code ListenableFuture}
	 * representing that task. The Future will return the Callable's result upon
	 * completion.
	 * @param task the {@code Callable} to execute (never {@code null})
	 * @return a {@code ListenableFuture} representing pending completion of the task
	 * @throws TaskRejectedException if the given task was not accepted
	 */
	<T> ListenableFuture<T> submitListenable(Callable<T> task);

}
```

这个接口中定义的方法仍然是接受Runnable或者Callable作为参数，但是返回值使用的是ListenableFuture，它是一个Future接口的扩展：

```java
public interface ListenableFuture<T> extends Future<T> {

	/**
	 * Register the given {@code ListenableFutureCallback}.
	 * @param callback the callback to register
	 */
	void addCallback(ListenableFutureCallback<? super T> callback);

	/**
	 * Java 8 lambda-friendly alternative with success and failure callbacks.
	 * @param successCallback the success callback
	 * @param failureCallback the failure callback
	 * @since 4.1
	 */
	void addCallback(SuccessCallback<? super T> successCallback, FailureCallback failureCallback);

}
```

它除了Future接口定义的功能外，还提供了增加回调的功能。这些回调在Future完成的那一刻会被立即调用。如果是使用AsyncResult返回结果的话，那么在执行addCallback的时候就会被调用：

```java
// Service中的Async方法，注意返回值是ListenableFuture接口类型。
@Async
public ListenableFuture<Integer> doSomethingListenableAsync() {
  AsyncResult<Integer> asyncResult = new AsyncResult<Integer>(42);
  asyncResult.addCallback(new ListenableFutureCallback<Integer>() {

    public void onSuccess(Integer result) {
      System.out.println(Thread.currentThread().getName() + " Callback, Result is: " + result);
    }

    public void onFailure(Throwable ex) {
      System.err.println(ex.getMessage());
    }
  });

  return asyncResult;
}
```

测试代码：

```java
@Test
public void testListenableAsync() throws InterruptedException, ExecutionException {
  ListenableFuture<Integer> future = service.doSomethingListenableAsync();
  System.out.println(Thread.currentThread().getName() + " Result is: " + future.get());
}
```

最终的输出是这样的：

```
SimpleAsyncTaskExecutor-1 Callback, Result is: 42
main Result is: 42
```

那么如何指定所需的Executor，而非默认的SimpleAsyncTaskExecutor呢？

通过前面讨论的qualifier这个概念。下面就来看看相关代码。

#### AsyncExecutionInterceptor

在这个类中：

```java
/**
 * This implementation is a no-op for compatibility in Spring 3.1.2. Subclasses may override to
 * provide support for extracting qualifier information, e.g. via an annotation on the given
 * method.
 * 
 * @return always {@code null}
 * @since 3.1.2
 * @see #determineAsyncExecutor(Method)
 */
@Override
protected String getExecutorQualifier(Method method) {
  return null;
}
```

因为兼容性的缘故，这个类并没有实现获取Executor Qualifier的功能。而是希望让子类实现，也就是让AnnotationAsyncExecutionInterceptor负责实现。

#### AnnotationAsyncExecutionInterceptor

```java
/**
 * Return the qualifier or bean name of the executor to be used when executing the given method,
 * specified via {@link Async#value} at the method or declaring class level. If {@code @Async} is
 * specified at both the method and class level, the method's {@code #value} takes precedence
 * (even if empty string, indicating that the default executor should be used preferentially).
 * 
 * @param method the method to inspect for executor qualifier metadata
 * @return the qualifier if specified, otherwise empty string indicating that the
 *         {@linkplain #setExecutor(Executor) default executor} should be used
 * @see #determineAsyncExecutor(Method)
 */
@Override
protected String getExecutorQualifier(Method method) {
  // Maintainer's note: changes made here should also be made in
  // AnnotationAsyncExecutionAspect#getExecutorQualifier
  Async async = AnnotatedElementUtils.findMergedAnnotation(method, Async.class);
  if (async == null) {
    async = AnnotatedElementUtils.findMergedAnnotation(method.getDeclaringClass(), Async.class);
  }
  return (async != null ? async.value() : null);
}
```

因此决定Qualifer的逻辑就是去@Async注解中寻找其value属性的值。

#### 使用基于ThreadPool的Executor

那么我们如何去定义一个不同于默认的Executor呢，还是通过JavaConfig：

```java
@Configuration
@EnableAspectJAutoProxy
@EnableAsync
public class SystemConfiguration {

  // ...

  @Bean(name = "tpExecutor")
  public Executor getThreadPoolExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(2);
    executor.setMaxPoolSize(2);
    executor.setQueueCapacity(500);
    executor.setThreadNamePrefix("TPExecutor-");
    executor.initialize();
    return executor;
  }
  
}  
```

然后在相应方法上使用带有属性值的@Async即可：

```java
@Async("tpExecutor")
public void doSomethingAsyncInTP() {
  System.out.println(Thread.currentThread().getName() + ": Async Biz in TP...");
}

// 相应测试方法：
@Test
public void testAsyncInTP() {
  service.doSomethingAsyncInTP();
}
```













