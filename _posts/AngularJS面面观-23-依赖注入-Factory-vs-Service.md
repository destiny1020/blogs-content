---
title: '[AngularJS面面观] 23. 依赖注入 --- Factory vs Service'
date: 2016-08-26 00:00:00
categories: [源码分析, AngularJS]
tags: [源码分析, AngularJS, 依赖管理]
---

据说99%的angular的初学者都会有一个疑问：`factory`和`service`到底有什么区别？什么情况该用`factory`，而什么情况又该用`service`呢？

比如这个Stackoverflow上的这个问题：[Service vs Factory](http://stackoverflow.com/questions/14324451/angular-service-vs-angular-factory)，又或者是这个问题：[Service vs Provider vs Factory](http://stackoverflow.com/questions/15666048/angularjs-service-vs-provider-vs-factory)。

这些问题都有热心答主回答的很棒了，能够解释清它们共同点，区别以及典型用法。因此本文就换个角度，从最根本的角度来看看这两个概念在源代码层次上是如何实现的。

<!-- More -->

## Factory

首先，我们来看看`module`上的`factory`方法，它是创建一个`factory`的入口方法：

```js
factory: invokeLaterAndSetModuleName('$provide', 'factory')

// 对于factory而言就是调用: invokeLaterAndSetModuleName('$provide', 'factory')
function invokeLaterAndSetModuleName(provider, method) {
  return function(recipeName, factoryFunction) {
    // 在factory函数上设置module名称
    if (factoryFunction && isFunction(factoryFunction)) factoryFunction.$$moduleName = name;

    // 将factory的定义置入任务队列
    invokeQueue.push([provider, method, arguments]);
    return moduleInstance;
  };
}
```

因此当我们使用该API创建一个`factory`时，该`factory`的"蓝图"可以表达成下面这个函数：

```js
// 假设我们创建一个名为aFactory的factory实例：module.factory('aFactory', function(){})
// 其中的recipeName就是aFactory
function(recipeName, factoryFunction) {
  if (factoryFunction && isFunction(factoryFunction)) factoryFunction.$$moduleName = name;
  invokeQueue.push(['$provide', 'factory', arguments]);
  return moduleInstance;
}
```

因此和前面讨论过的`provider`和`constant`一样，`factory`最终也是由`$provide.factory`方法生产制造出来的(`provider`对应`$provide.provider`，`constant`对应`$provide.constant`)：

```js
function factory(name, factoryFn, enforce) {
  return provider(name, {
    $get: enforce !== false ? enforceReturnValue(name, factoryFn) : factoryFn
  });
}
```

从上面这段代码就可以看出现端倪了。在`factory`函数的实现内部，它利用了`provider`实现其功能。如果对于provider有疑问可以先看看这篇文章 - [依赖注入 --- Provider是个啥](http://blog.csdn.net/dm_vincent/article/details/52137733)。

我们知道`provider`的目的就是定义依赖应该如何被生成。而定义这个任务的核心就是其内部的$get方法。而`factory`的目的也是为了定义一个依赖，从目的上而言它和`provider`的目的不谋而合。因此，`factory`函数在内部创建了一个符合`provider`契约的对象，该对象的`$get`方法实际上就是参数中的`factoryFn`，而这个`factoryFn`则是我们定义的。因此调用`$get`方法就是再调用`factoryFn`，返回的值也是我们在`factoryFn`中定义的返回值。

只不过`factory`函数的第三个参数对具体的`$get`方法有一点点影响，如果不将它显式地设置成`false`(必须是false，undefine，null这些都不算)，那么就会保证该`factory`必须是返回值的，否则会抛出异常：

```js
function enforceReturnValue(name, factory) {
  return function enforcedReturnValue() {
    // 实例注入器完成调用
    var result = instanceInjector.invoke(factory, this);
    // 如果返回值为undefined，抛出异常
    if (isUndefined(result)) {
      throw $injectorMinErr('undef', "Provider '{0}' must return a value from $get factory method.", name);
    }
    return result;
  };
}
```

由于我们平常使用`factory`很少关注第三个参数，因此事实上我们总是确保了`factory`必须要返回点东西作为其返回值。

分析到这里，`factory`的实现其实没什么新玩意，算是新瓶装旧酒吧，本质上还是利用了`provider`来完成其功能。值得注意的地方是，`factory`必须要返回一个值，由于`factory`是通过实例注入器来调用的，因此它的返回值最终会被保存到实例注入器的cache中，所以以后不管什么地方通过需要它，拿到的都是同一份实例，也就是说这个`factory`其实是一个单例对象，在整个angular应用中都只有一份。因此`factory`中保存的数据是全局唯一的，我们可以把它当作一个用于保存全局数据的地方。

## Service

看完了`factory`的实现，我们再来看看`service`。`service`这个名字其实挺泛的，什么东西都可以叫做一个`service`，`factory`难道就不是一个`service`吗？下面是它相关的代码：

```js
// Module API
service: invokeLaterAndSetModuleName('$provide', 'service')


```

和同样定义在Module API中的`factory`非常相似，只不过第二个参数从'factory'变成了'service'。这个`service`指向的函数如下所示：

```js
// Injector内部的service函数
function service(name, constructor) {
  return factory(name, ['$injector', function($injector) {
    return $injector.instantiate(constructor);
  }]);
}
```

注意`service`函数的第二个参数名是`constructor`，也隐含着它和`factory`的不同。即我们传入的函数是会被当作一个构造函数来调用的，其内部的实现也证实了这一点：

```js
// 调用的是注入器的instantiate方法，而非invoke方法
return $injector.instantiate(constructor);
```

另外一个非常非常有意思的地方是，`service`的内部利用了`factory`！它实际上还是创建了一个`factory`，只不过这个`factory`的任务就是实例化我们定义的`constructor`：

```js
// 内部利用factory完成service的功能
return factory(name, ['$injector', function($injector) {
  return $injector.instantiate(constructor);
}]);
```

在内部创建的`factory`中，使用基于数组的风格完成了注解的声明(如果不清楚注解是什么，可以先看这篇文章：[依赖注入 --- 注解的定义与实现](http://blog.csdn.net/dm_vincent/article/details/52081180))。它依赖于`$injector`服务，那么这个`$injector`是指的实例注入器呢，还是provider注入器呢？

首先我们知道`service`，`factory`这一类对象在需要被实例化的时候，实例化它们的请求都是通过实例注入器发起的。当实例注入器发现实例化一个`service`的时候，还需要首先处理掉它的依赖，也就是上述代码中出现的`$injector`时，就会首先去实例注入器自身的cache中寻找，发现没有名为"`$injector`"的对象后，就会去寻求provider注入器的帮助(关于这个流程的具体细节，可以参考[依赖注入 --- instance注入器以及provider注入器](http://blog.csdn.net/dm_vincent/article/details/52140633))，这个时候就会使用键值"`$injectorProvider`"去provider注入器中寻找，而恰好在创建注入器的时候就已经定义好这个对象了：

```js
// providerSuffix就是"Provider"
// protoInstanceInjectorj就是实例注入器
providerCache['$injector' + providerSuffix] = { $get: valueFn(protoInstanceInjector) };
```

所以最终`$injector`对应的就是实例注入器，使用的是实例注入器来完成`service`构造函数的实例化。和`factory`的一点不同在于，`service`并不需要定义一个明确的返回值，因为注入器总是会将`service`的函数当作构造函数创建出一个新对象出来作为依赖。而`factory`则是自己声明返回的对象作为依赖。

## 简单总结

Factory和Service是我们经常用的两个API，它们都能够帮助我们定义依赖注入需要的对象。它们的共同点和不同点简单总结如下：

### 共同点

1. 创建的对象都是单例对象。这一点是依赖注入机制确保的，因为一旦一个对象被创建完毕了，它总是会被保存到实例注入器的cache中，下次查找这个对象时总是能够从cache中找到它，因此就没有必要再次创建了。
2. 在底层都创建了一个临时的provider。这一点体现在factory的实现中，而service又在内部依赖于factory。因此它们最终都依赖于这个临时的provider。


### 不同点

1. 被注入器调用的方式不同。factory通过`$injector.invoke`调用；而service则通过`$injector.instantiate`调用。
2. 声明函数的调用方式不同。这一点其实和第一点是对应的。factory声明函数被当作一个普通的函数调用，即invoke方式；而service声明函数则是被当作构造函数调用，即instantiate方式。
3. 是否需要返回值。factory必须要有一个返回值作为依赖对象；service没有这个必要，调用构造函数实例化得到的对象即为依赖对象。


弄清楚了源代码是如何实现factory和service的，再去看看文章开头处提到的那些问题，是不是觉得一切豁然开朗呢？



