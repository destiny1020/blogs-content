---
title: '[AngularJS面面观] 24. 依赖注入 --- Value以及Decorator'
date: 2016-08-28 18:55:00
categories: [源码分析, AngularJS]
tags: [源码分析, AngularJS, 依赖管理]
---

module中定义的高层API现在已经介绍的差不多了，本文就把后面剩下的几个能介绍的先介绍了(不能介绍的还有蛮多的，比如filter，controller，directive，这些使我们后面讨论的内容，敬请期待 :) )。

和依赖注入关系比较紧密的剩下2个方法分别是value和decorator。

## Value

在angular中，比较常见的问题除了service，factory和provider三者之间有何区别和联系之外(想知道答案就去看[依赖注入 --- Factory vs Service](http://blog.csdn.net/dm_vincent/article/details/52200952))，还有一个问题也算比较热门。那就是constant和value有什么区别？

光从定义方式上看，它们两者好像是没有什么区别：

<!-- More -->

```js
module.constant('a', 'aConstant');
module.value('b', 'bValue');
```

它们的定义方式都很简单，就是一个键值对。然而angular为什么要弄两个API出来呢？让我们从代码层面上看看：

```js
// value在module中的定义
value: invokeLater('$provide', 'value'),

// constant在module中的定义
constant: invokeLater('$provide', 'constant', 'unshift')
```

可以发现，它们的区别在于一个对应注入器内部的value函数，另一个对应constant函数：

```js
function value(name, val) { 
  return factory(name, valueFn(val), false); 
}

// factory函数的定义
function factory(name, factoryFn, enforce) {
  return provider(name, {
    $get: enforce !== false ? enforceReturnValue(name, factoryFn) : factoryFn
  });
}

// valueFn的定义
function valueFn(value) {
  return function valueRef() {
    return value;
  };
}
```

所以value本质上也是基于factory来实现的，通过valueFn将一个值封装成一个返回它自身的函数。而这里也利用到了factory函数的第三个参数：enforce。将它设置为false表示value可以被定义为undefined，也就是说下面的定义是合法的，尽管我们不会无聊到去这样做：

```js
module.value('a', undefined);
```

正式因为value本质上也是依赖于factory的，所以value和constant的区别也就一目了然了。constant对于两个注入器都是可见的，因为在constant的实现中将constant的值同时放入到了两个注入器的缓存中。而value则不然，它的值仅对于实例注入器可见。所以在provider的构造函数以及config队列定义的任务中可以将constant声明为依赖，但是不可将value声明为依赖。因为provider的构造函数以及config队列是通过provider注入器来调用的。

但是在实际开发中，用到value的场合并不太多。至少对于我而言很少用到它，因为value提供的功能用constant基本上都能够满足。而constant由于其对两个注入器都是可见的，所以使用起来更方便，只要支持依赖注入的地方就可以使用，不用区分实例注入器和provider注入器。

## Decorator

装饰器算是依赖注入部分最后一个要介绍的概念。顾名思义，装饰器实现了[装饰器模式(Decorator Pattern)](https://en.wikipedia.org/wiki/Decorator_pattern)。利用这种模式能够给已经存在的对象附加一些额外的功能或者改变现有的功能，也就是"装饰"它。

那么在angular依赖注入这一上下文环境中，装饰器的作用又是什么呢？

在开发angular应用的时候，我们都引用过其它的第三方模块。引入第三方模块，从依赖注入的观点来看，就是将第三方模块中定义的各种服务都注册一遍，然后在需要的时候将这些服务实例化，放到注入器的缓存中保管并将它们注入到被需要的地方。绝大多数时候第三方的服务都能够很好的工作，但是偶尔我们也想改动一下它们使之能够更好的契合业务需求。这个时候有两条路可以选：

1. fork这个第三方模块然后直接改动其源代码
2. 使用decorator对它进行改动

毫无疑问，使用第一种方式在多数情况下并不可取，fork一个模块的代价可能是放弃这个模块后续的版本，毕竟你的改动让这个模块和它后续的版本可能不再兼容。所以为了解决这个问题，angular也提供了装饰器模式的实现decorator：

```js
decorator: invokeLaterAndSetModuleName('$provide', 'decorator')
```

可见decorator在定义方式上和其它的factory，service并没有什么本质区别。唯一的区别就是它的"蓝图"是`$provide`中的decorator函数：

```js
function decorator(serviceName, decorFn) {
  // 直接从provider注入器的缓存中取得相应provider
  var origProvider = providerInjector.get(serviceName + providerSuffix),
      orig$get = origProvider.$get;  // 将原$get方法保存起来，准备覆盖

  // 覆盖$get方法
  origProvider.$get = function() {
    // 得到未经装饰的实例对象
    var origInstance = instanceInjector.invoke(orig$get, origProvider);

    // 利用实例注入器调用装饰器函数，未经装饰的实例以$delegate参数传入
    return instanceInjector.invoke(decorFn, null, {$delegate: origInstance});
  };
}
```

首先来看看函数的参数是什么意思：

serviceName：表示的是需要装饰的服务名字。
decorFn：装饰函数，用以实现具体的装饰行为。

从decorator函数的实现来看，需要明确两个问题：

1. 为什么可以直接拿到service相对应的provider？
2. invoke方法的第三个参数如何工作？

先来回答第一个问题。

decorator函数的第一个参数serviceName其实是一个泛指，并不一定非要是一个service，其实factory，value等等都是可以的(因为它们都有对应的provider；而constant不行，因为它并没有对应的provider)。而我们知道不管是service也好，factory也好，还是value也好。它们最终都是依赖于provider的，而注入器内部的provider函数实现如下：

```js
function provider(name, provider_) {
  assertNotHasOwnProperty(name, 'service');
  if (isFunction(provider_) || isArray(provider_)) {
    provider_ = providerInjector.instantiate(provider_);
  }
  if (!provider_.$get) {
    throw $injectorMinErr('pget', "Provider '{0}' must define $get factory method.", name);
  }

  // 将创建得到的provider以name + "Provider"的键值命名方式存入到provider注入器缓存中
  return providerCache[name + providerSuffix] = provider_;
}
```

该函数的最后一行是关键：将创建得到的provider以name + "Provider"的键值命名方式存入到provider注入器缓存中。所以`origProvider = providerInjector.get(serviceName + providerSuffix)`是可以正常工作的。

下面回答第二个问题。其实这个问题在以前介绍注入器的基础知识的时候就已经讨论过了，这个参数的名字叫做locals，它用来提供额外的依赖或者是覆盖现有的依赖。下面再回顾一下invoke的调用过程就算是加深印象吧。

拿到了provider，就相当于拿到了创建一个依赖对象的"蓝图"，在装饰器中就可以先将它创建出来，然后再随心所欲的修改它了。修改时调用的：

```js
// 第三个参数是locals
instanceInjector.invoke(decorFn, null, {$delegate: origInstance})

// 注入器的invoke函数实现
function invoke(fn, self, locals, serviceName) {
  if (typeof locals === 'string') {
    serviceName = locals;
    locals = null;
  }

  // 将声明的参数注入到函数中
  var args = injectionArgs(fn, locals, serviceName);
  if (isArray(fn)) {
    fn = fn[fn.length - 1];
  }

  if (!isClass(fn)) {
    return fn.apply(self, args);
  } else {
    args.unshift(null);
    return new (Function.prototype.bind.apply(fn, args))();
  }
}

// 准备依赖对象
function injectionArgs(fn, locals, serviceName) {
  var args = [],
      $inject = createInjector.$$annotate(fn, strictDi, serviceName);

  for (var i = 0, length = $inject.length; i < length; i++) {
    var key = $inject[i];
    if (typeof key !== 'string') {
      throw $injectorMinErr('itkn',
              'Incorrect injection token! Expected service name as string, got {0}', key);
    }

    // 如果locals中含有key相同的属性，就使用locals中的属性作为依赖
    args.push(locals && locals.hasOwnProperty(key) ? locals[key] :
                                                     getService(key, serviceName));
  }
  return args;
}
```

这里完成了将未经修改的实例以`$delegate`传入到装饰器函数中。比如下面这段代码就实现了一个简单的装饰，给一个factory的返回实例增加了一个字段：

```js
module.factory('aFactory', function() {
  return {
    a: 'aConstant'
  };
});

module.decorator('aFactory', function($delegate) {
  $delegate.b = 'bConstant';
});

module.run(function(aFactory) {
  console.log(aFactory.a);  // 输出：aConstant
  console.log(aFactory.b);  // 输出：bConstant
});
```

decorator使用的场合并不是那么多，但是了解它的用法有时候能够帮上大忙。

---

至此，angular中有关依赖注入的讨论就告一段落了。一共有好些篇文章来介绍这个特性，毕竟这个特性是angular中一个不可或缺同时也是非常吸引人的特性。它可以说是angular框架的中枢神经，在它的帮助下，才得以实现诸多强大的特性，比如前面介绍过的scope，以及未来要介绍的directive等等。

文章写的可能存在一些问题，如果有任何疑问欢迎提出来一起探讨。

谢谢。

