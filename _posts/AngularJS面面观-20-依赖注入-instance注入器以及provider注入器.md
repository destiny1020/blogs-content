---
title: '[AngularJS面面观] 20. 依赖注入 --- instance注入器以及provider注入器'
date: 2016-08-13 22:08:00
categories: [源码分析, AngularJS]
tags: [源码分析, AngularJS, 依赖管理]
---

本文就来解答上一篇文章留下的疑问，为什么在注入器也分成了`instance`注入器和`provider`注入器。这两种注入器的工作原理是怎么样的。

## 总体结构

为此我特别准备了一张图来描述一下angular注入器的工作流程和原理，如下所示。

![angular注入器工作流程和原理](http://img.blog.csdn.net/20160807001803717)

<!-- More -->

这张图的顶部是外部调用的入口，即通过angular暴露给外部的`$injector`服务。关于`$injector`服务中含有的五个方法，在[$injector服务](http://blog.csdn.net/dm_vincent/article/details/52076335)中已经介绍过了，在本文中就不再赘述了。还不清楚`$injector`服务是个啥的同学可以先去看看那篇文章。

那么暴露给外部使用的`$injector`是哪种注入器呢？从图中可以很清晰的看出，暴露的是实例注入器(Instance Injector)。所谓的实例注入器，就是用来保存各种被实例化了的托管对象。这些实例在实例化之后会被保存在它对应的一个叫做实例注入器缓存的地方，也就是图中的cache(instanceCache)所指向的区域。下面我们就来看看实例注入器的相关代码。

### instance注入器

```js
instanceCache = {},
protoInstanceInjector =
    createInternalInjector(instanceCache, function(serviceName, caller) {
      var provider = providerInjector.get(serviceName + providerSuffix, caller);
      return instanceInjector.invoke(
          provider.$get, provider, undefined, serviceName);
    }),
instanceInjector = protoInstanceInjector;

// createInternalInjector的函数签名
function createInternalInjector(cache, factory) { 
  // ......
}

// factory在getService函数中被用到
function getService(serviceName, caller) {
  if (cache.hasOwnProperty(serviceName)) {
    if (cache[serviceName] === INSTANTIATING) {
      throw $injectorMinErr('cdep', 'Circular dependency found: {0}',
                serviceName + ' <- ' + path.join(' <- '));
    }
    return cache[serviceName];
  } else {
    try {
      path.unshift(serviceName);
      cache[serviceName] = INSTANTIATING;
      // 注意下面这一行代码的行为，调用factory得到实例然后保存并返回
      return cache[serviceName] = factory(serviceName, caller);
    } catch (err) {
      if (cache[serviceName] === INSTANTIATING) {
        delete cache[serviceName];
      }
      throw err;
    } finally {
      path.shift();
    }
  }
}
```

注入器本身的实例是通过调用`createInternalInjector`函数得到。该函数需要接受两个参数，第一个参数`cache`表示注入器使用的缓存是哪一个。第二个参数`factory`是一个函数，当需要的实例不存在时就会调用这个函数来进行实例化，得到的对象会被放入到第一个参数所指定的缓存中。

那让我们来看看实例注入器是如何声明这个`factory`函数的：

```js
// providerSuffix实际上就是一个字符串：Provider
var providerSuffix = 'Provider';

createInternalInjector(instanceCache, function(serviceName, caller) {
  var provider = providerInjector.get(serviceName + providerSuffix, caller);
  return instanceInjector.invoke(provider.$get, provider, undefined, serviceName);
}),
```

很明显的是，在实例注入器的`factory`函数中发生了和`provider`注入器的互动。它会去`provider`注入器中寻找一个`provider`对象，然后通过实例注入器来调用`provider`对象上的`$get`方法，来最终得到需要的对象实例。这个过程在前文的总体结构图中也反映出来了，实例注入器指向`provider`注入器的箭头表达的就是这个意思，即它会将托管对象实例化的工作委托给保存在`provider`注入器中的某个`provider`。具体而言，是委托给了`provider`对象上的`$get`方法。该方法最终会被实例注入器通过`invoke`方法进行调用。既然`$get`方法是通过注入器的`invoke`方法而调用，因此在`$get`方法中开发人员也可以声明它所需要的各种依赖。

### provider注入器

简要分析了实例注入器之后，下面来看看另一个主角 - `provider`注入器。
它的创建过程是这样的：

```js
providerCache = {
  $provide: {
      provider: supportObject(provider),
      factory: supportObject(factory),
      service: supportObject(service),
      value: supportObject(value),
      constant: supportObject(constant),
      decorator: decorator
    }
},
providerInjector = (providerCache.$injector =
  createInternalInjector(providerCache, function(serviceName, caller) {
    if (angular.isString(caller)) {
      path.push(caller);
    }
    throw $injectorMinErr('unpr', "Unknown provider: {0}", path.join(' <- '));
  }));
```

首先，`providerCache`在初始化的时候就已经有一个`$provide`对象了。这个对象中定义了我们很熟悉的`provider`，`factory`，`service`等一系列我们能够在`module`上定义的服务类型。至于这些方法的是做什么用的，如何使用的，我们后面会专门进行分析。本文暂且先专注于注入器实现的总体结构和执行流程。

然后还是通过`createInternalInjector`函数来完成`provider`注入器的创建工作。创建得到的注入器也会被保存到`providerCache`中。而该注入器提供的`factory`函数的定义也特别简单。就是记录一下调用的顺序，然后抛出异常。为什么直接抛出找不到`provider`的异常？因为如果在`providerCache`中都找不到我们需要的`provider`对象，就没有必要继续寻找下去了，毕竟只有两个注入器不是嘛。实例注入器可以委托`provider`注入器，但是`provider`注入器可没别的注入器依赖。这也是为什么`providerCache`在初始化的时候就已经有一个`$provide`对象的原因。因为这个`$provide`对象上定义的方法可不简单，它们的职责就是创建我们开发人员在`module`上定义的各种服务。如果没有它们的存在，那每次定义服务都会抛出`Unknown Provider`异常了不是吗。

另外值得一提的是，在创建了实例注入器和`provider`注入器之后，还有这么一行代码：

```js
providerCache['$injector' + providerSuffix] = { $get: valueFn(protoInstanceInjector) };

// valueFn的定义
function valueFn(value) {return function valueRef() {return value;};}
```

它将实例注入器的通过`$injectorProvider`这个键值给注册到了`providerCache`中去。
所以，当我们在自定义的`service`，`factory`，`controller`中指定需要`$injector`时，得到的都会是实例注入器，而非`provider`注入器。这个过程也很清晰，当你需要`$injector`时，实例注入器会委托给`provider`注入器，`provider`注入器去寻找一个名为`$injectorProvider`的`provider`对象，并调用其`$get`方法返回真正的实例。

### 为什么要这样实现

看清楚了angular注入器的实现原理和流程，我们来思考一下为什么它会这么设计。这个问题也许只有开发这个功能的人才能最终解释清楚。但是这一点是很明显的：

**隐藏不必要的信息，将功能的入口单一化**。也就是所谓的门面模式(Facade Pattern)。这个门面就是angular中的`module`。我们注册各种服务都是使用的`module`上的对应方法，作为开发人员只需要了解清除module上的各种方法的使用规则就可以了，没有知道底层还有个`$injector`服务的必要。就算开发的功能比较复杂，需要使用到`$injector`服务了，也确实没有必要需要知道这个`$injector`服务实际上是由两个`injector`联合提供的。这也是软件开发的基本规则之一，即使软件提供的功能很复杂，对外暴露的也要尽可能地简单易懂。否则你这个软件还怎么用，难道要用之前得看一遍源代码理解其内部实现才行吗。

---

但是学习angular中注入器的设计和实现也是一件非常有意思的事情，真的是开拓了眼界。一个框架的重要功能原来是这么实现的，一点一点地去分析去思考也是能够看懂背后的逻辑的，毕竟再复杂的程序也是一行一行代码垒起来的。


