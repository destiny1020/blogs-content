---
title: '[AngularJS面面观] 19. 依赖注入 --- Provider是个啥'
date: 2016-08-13 22:07:00
categories: [源码分析, AngularJS]
tags: [源码分析, AngularJS, 依赖管理]
---

在前面介绍angular中依赖注入相关的概念和细节时，非常多次提到了`provider`这个概念，每次提到都会让大家再等等，再等等。现在再也等不了啦，从本篇文章开始就会陆续介绍`provider`和一些基于`provider`的高层方法，比如`service`，`factory`等等。

## provider是什么？

### 通过对象声明provider

首先，我们来看看`provider`是什么。在angular中，`provider`就是知道如何处理依赖关系的一类对象。为啥说是一类对象呢，因为只要一个对象实现了`provider`规定的契约，那么该对象就可以被称作`provider`。这个契约也非常的简单：提供一个带有返回值的`$get`方法即可。

<!-- More -->

比如下面的对象就是符合要求的`provider`：

```js
{
  $get: function() {
    return 'aConstant';
  }
}
```

上面的对象定义了一个`$get`方法，该方法返回了一个字符串作为返回值。

现在，我们定义了一个最简单的`provider`，该如何把这个`provider`注册到模块中呢？没错，还是通过`module`实例提供的`provider`方法：

```js
var app = angular.module('test', []);

// 通过provider提供一个常量的获取方法
app.provider('a', {
  $get: function() {
    return 'aConstant';
  }
});

// 直接通过constant定义一个常量
app.constant('b', 'bConstant');
```

在上述代码中，我们分别使用`provider`和`constant`方法定义了常量。比较一下两种方法，看看他们有什么区别？可能会有同学说不就是定义一个常量，有必要大费周章吗？确实，使用`provider`来定义常量从步骤上而言会繁琐一些，但是它也增加了代码的灵活性不是吗？在计算机领域，有一句名言："计算机科学领域的任何问题都可以通过增加一个间接的中间层来解决"。这句话放在这里也同样试用，如果因为种种原因常量的定义不能在直接在代码中确定，那么将这个定义的过程延迟到`$get`方法中不就是一种合理的解决方案吗？这里`provider`的作用就是提供这一间接的中间层。

现在我们回头看看`module`类型中是如何定义`provider`方法的：

```js
provider: invokeLaterAndSetModuleName('$provide', 'provider')

// invokeLaterAndSetModuleName本身也是一个函数，返回另一个函数
function invokeLaterAndSetModuleName(provider, method) {
    return function(recipeName, factoryFunction) {
      if (factoryFunction && isFunction(factoryFunction)) factoryFunction.$$moduleName = name;
      invokeQueue.push([provider, method, arguments]);
      return moduleInstance;
    };
  }
});
```

当然，为了使`provider`能够起到作用，它的`$get`方法也是能够声明其依赖的，比如这样：

```js
var app = angular.module('test', []);

// 通过provider提供一个常量的获取方法
app.provider('a', {
  // 声明依赖关系，最终得到的a常量为"aConstantbConstant"
  $get: function(b) {
    return 'aConstant' + b;
  }
});

// 直接通过constant定义一个常量
app.constant('b', 'bConstant');
```

### 函数的柯里化

看上去很奇怪，一个函数返回了另外一个函数。其实这里涉及到一个叫做"[函数的柯里化](http://baike.baidu.com/link?url=AgQxcMsyJnEZqrImRjKcxkUu652-DRfrkab23OH7fdVfoXi5fadI1coYM-AFN3XQ1au99I4BE7s2EAt1eeV3Ua)"的概念，是一种常见的将接受多个参数的函数转化为只接受一个或者有限几个的函数的方法。为了分析这个问题，我们不妨假设一下现在并没有`invokeLaterAndSetModuleName`这个函数。那么要完成既定任务，`provider`方法对应的函数签名就必须变成这样了：

```js
function(provider, method, recipeName, factoryFunction) {}
```

四个参数有点多了，而且复用性不佳。使用了柯里化将四个参数的函数转换为两个参数的函数，同时还能够在函数内部完成一些额外的任务，比如在`invokeLaterAndSetModuleName`函数中就还完成了一项设置`module`名字的任务。最后，函数返回的是`moduleInstance`，意味着它能够支持所谓的"链式编程"。就是可以连续地在`module`上定义多个服务，比如这样：

```js
angular.module('test', []).constant('a', 'aConstant').constant('b', 'bConstant');
```

定义了`provide`r后，在什么时候会运行呢？在`module`被加载的时候，会执行下面这段代码(关于`module`加载和任务队列，可以参考这篇文章[初识注入器](http://blog.csdn.net/dm_vincent/article/details/52015678))：

```js
function runInvokeQueue(queue) {
  var i, ii;
  for (i = 0, ii = queue.length; i < ii; i++) {
    var invokeArgs = queue[i],
          // 得到$provide对象
          provider = providerInjector.get(invokeArgs[0]);
    
    // invokeArgs的对应关系：[0] - '$provide', [1] - 'provider', [2] - [providerName, providerDef] 
    provider[invokeArgs[1]].apply(provider, invokeArgs[2]);
  }
}
```

执行任务队列的关键就在于上述for循环中的最后一行：`provider[invokeArgs[1]].apply(provider, invokeArgs[2]);`，它实际上的调用的是`$provide`对象上的`provider`方法：

```js
function provider(name, provider_) {
  // 确保name不等于hasOwnProperty
  assertNotHasOwnProperty(name, 'service');
  // 如果提供的是provider_是函数或者数组，直接使用injector实例化
  if (isFunction(provider_) || isArray(provider_)) {
    provider_ = providerInjector.instantiate(provider_);
  }
  // 如果provider_没有提供$get方法，抛出异常
  if (!provider_.$get) {
    throw $injectorMinErr('pget', "Provider '{0}' must define $get factory method.", name);
  }
  // 将provider_放入到providerCache中
  return providerCache[name + providerSuffix] = provider_;
}
```

可以发现，最终`provider`会调用`providerInjector.instantiate`方法进行实例化。而实例化的过程我们在之前的文章中已经介绍过了，它能够通过注解信息找到需要的被托管对象。需要注意的是，`instantiate`方法的调用只有当`provider_`参数是函数或者数组类型时才会发生。像我们之前那样通过`object`的方式定义`provider`并不会触发`instantiate`的调用。这样只会将该`object`放入到`providerCache`中供未来使用。

### 通过构造器函数声明provider

那么，从代码中可以看出`provider`的定义方式并非只有通过`object`一种方法，还能够通过构造器函数或者数组的形式定义：

```js
var app = angular.module('test', []);

// 通过provider提供一个常量的获取方法，使用构造器函数
app.provider('a', function AProvider() {
  this.$get = function() {
    return 'aConstant';
  };
});
```

函数的名字并非一定要是`AProvider`，任何别的什么名字都可以。但是这里作为一种约定，使用它能够让代码的可读性更好。可以看到，只要这个`AProvider`类型中提供了一个名为`$get`的方法即可。这一点和使用`object`来定义`provider`是一样一样的。

所以通过构造器函数的方式来定义`provider`的话，就会执行上述代码中的`providerInjector.instantiate(provider_)`了，而我们已经知道`instantiate`方法是有能力来处理依赖问题的。因此在`provider`的构造器函数中，可以定义需要依赖的函数，比如这样：

```js
var app = angular.module('test', []);

app.constant('b', 'bConstant');

// 通过provider提供一个常量的获取方法，使用构造器函数并声明依赖
app.provider('a', function AProvider(b) {
  this.$get = function() {
    // 那么最终a常量的定义就是： aConstantbConstant
    return 'aConstant' + b;
  };
});
```

### 两种provider声明方式的区别

现在，我们目前有两种方式来定义`provider`：
1. 通过对象的方式，该对象需要有一个类型为函数的`$get`属性。
2. 通过构造器函数的方式，该函数对象内部有一个`$get`方法。

那么这两种方式是否是一样的，在任何场合下都能够互换呢？答案是否定的，其实答案在上述介绍`provider`函数的源码时就已经揭晓了，让我们再来看看该函数的定义：

```js
function provider(name, provider_) {
  // 确保name不等于hasOwnProperty
  assertNotHasOwnProperty(name, 'service');
  // 如果提供的是provider_是函数或者数组，直接使用injector实例化
  if (isFunction(provider_) || isArray(provider_)) {
    provider_ = providerInjector.instantiate(provider_);
  }
  // 如果provider_没有提供$get方法，抛出异常
  if (!provider_.$get) {
    throw $injectorMinErr('pget', "Provider '{0}' must define $get factory method.", name);
  }
  // 将provider_放入到providerCache中
  return providerCache[name + providerSuffix] = provider_;
}
```

**区别就在于懒加载是否能够实现。**

第一种使用对象的`provider`声明方式并不会导致`instantiate`方法的执行。而第二种使用构造器函数的方式则会触发`instantiate`方法。因此，第一种方式能够实现被托管对象的懒加载，而第二种方式则不能，如果声明为构造器函数则总是会急不可耐地调用`instantiate`方法完成它的所有依赖和它本身的实例化。所谓懒加载，就是当真正地需要某个被注入器托管的对象时才会去创建。而这个真正去创建的过程就是`invoke`/`instantiate`方法去做的事情。关于懒加载的相关讨论，可以参考这一篇文章[依赖注入 --- 注入器中如何管理对象](http://blog.csdn.net/dm_vincent/article/details/52073838)。

那么是谁来实例化这个我们定义的构造器函数形式的`provider`呢？在源码中是一个叫做`providerInjector`的注入器，它和我们通常所说的注入器是一个东西吗？从源代码来看：

```js
function createInjector(modulesToLoad, strictDi) {
  strictDi = (strictDi === true);
  var INSTANTIATING = {},
      providerSuffix = 'Provider',
      path = [],
      loadedModules = new HashMap([], true),

      // provider cache以及provider injector
      providerCache = {
        // ......
      },
      providerInjector = (providerCache.$injector =
          createInternalInjector(providerCache, function(serviceName, caller) {
            if (angular.isString(caller)) {
              path.push(caller);
            }
            throw $injectorMinErr('unpr', "Unknown provider: {0}", path.join(' <- '));
          })),

      // instance cache以及instance injector
      instanceCache = {},
      protoInstanceInjector =
          createInternalInjector(instanceCache, function(serviceName, caller) {
            var provider = providerInjector.get(serviceName + providerSuffix, caller);
            return instanceInjector.invoke(
                provider.$get, provider, undefined, serviceName);
          }),
      instanceInjector = protoInstanceInjector;

  // ......

  // 返回的是instance injector
  return instanceInjector;

  // 其它内部函数...
}
```

答案很明确，并不是一个东西。返回的是`instance`注入器，而`provider`注入器只是存在于内部的另外一个注入器。

真是一个问题接着一个问题，解决了一个问题又引出了另外一个问题。不过这也正是程序开发的有意思之处。世界这么复杂，怎么可能三言两语就讲的明白呢。

那么我们的问题就转变成了为何要如此设计呢，`instance`注入器和`provider`注入器有什么区别和联系，它们之间存在交互行为吗？这些问题留待下一篇文章解答。
