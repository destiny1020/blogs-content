---
title: '[AngularJS面面观] 21. 依赖注入 --- constant到底如何而来'
date: 2016-08-19 23:11:00
categories: [源码分析, AngularJS]
tags: [源码分析, AngularJS, 依赖管理]
---

在上一篇文章中，我们终于见到了angular中依赖注入的总体结构图。从这幅图中我们可以知道在angular内部是有两个注入器协同工作来实现我们习以为常的依赖注入特性的。

![angular注入器工作流程和原理](http://img.blog.csdn.net/20160807001803717)

结合上图简单回顾一下angular依赖注入的组成和工作流程。

<!-- More -->

首先，在台面上的注入器名为实例注入器(Instance Injector)，它里面含有一个名为实例缓存(Instance Cache)的字典对象，该缓存的作用是保存被托管的对象，每个被注入器实例化得到的对象都会被保存在其中。所谓的依赖注入，实际上就是从实例注入器的缓存中拿去我们需要的对象。当然，凡事都有第一次，当我们需要的对象并不在该缓存中时，也就是说该对象还没有被实例化。那么这个时候实例注入器就要像provider注入器求援了。因为在这个provider注入器中保存的都是用于实例化对象的"菜谱"，而这些"菜谱"就定义在了每个`provider`对象的`$get`方法中。因此，调用该对象的`$get`方法，实例注入器就能够获取到需要的对象，接下来就是保存该对象到实例注入器的缓存中并将该对象注入到需要它的地方。

---

## 基于provider的高层API

然而，我们在真正地应用angular框架来完成我们的业务逻辑时，直接使用`$injector`以及定义各种`provider`并非不行，而是太底层了。所以angular给我们封装了各种各样的服务，比如`factory`，`service`，`controller`，`value`，`constant`等等。别看它们一个个名字洋气得很，其实万变不离其宗，在幕后都有一个`provider`在默默的支持着。所以，从本文开始我们将系统地讨论这些封装好了的服务，揭开它们华丽的外衣，还原其本质。既然提到了这些服务本质上都是基于`provider`的，所以首先你就应该搞清楚`provider`是怎么一回事，可以参考这篇文章[依赖注入 --- Provider是个啥](http://blog.csdn.net/dm_vincent/article/details/52137733)，里面对`provider`作出了一些介绍。

### constant的一生

按照惯例，还是先挑软柿子捏，最简单的服务非`constant`莫属。在前面的文章中，我们也一直拿`constant`作为例子来讨论依赖注入的工作原理和实现细节。但是一直都没有正儿八经地看看`constant`是如何实现的。所以，我们就先来看看`constant`的一生：它是如何被定义，如何被创建，又是如何被注入的。

#### 定义

我们都知道要声明`constant`，使用的就是`module.constant`方法：

```js
constant: invokeLater('$provide', 'constant', 'unshift')

function invokeLater(provider, method, insertMethod, queue) {
  if (!queue) queue = invokeQueue;
  // 还是利用柯里化将多个参数的函数转换为少数参数的函数
  return function() {
    // arguments才是我们在声明constant时实际传入的参数
    queue[insertMethod || 'push']([provider, method, arguments]);
    return moduleInstance;
  };
}

// 将上面代码还原一下，constant的实际定义是这样的：
constant: function() {
  invokeQueue['unshift'](['$provide', 'constant', arguments]);
  return moduleInstance;
}
```

这里仍然使用了JavaScript中一个被经常使用的模式，即函数的"柯里化"，它的主要作用是减少函数的参数数量，是一个函数式编程中经常会被使用到的模式。

所以对于`constant`的定义，就是往任务队列里面增加一条记录。只不过，它这里用到的`insertMethod`是`unshift`而并非默认的`push`，这一点值得留意，它将`constant`的定义放在了任务队列的头部。所以不管应用程序是以何种顺序来定义`constant`的，当注入器加载模块的时候总是会优先执行代表`constant`的任务。为何需要优先执行`constant`任务呢？其实原因很简单：因为`constant`很简单不是嘛。它又不需要依赖别的被托管对象，因此一开始就执行它们准没错！

那么我们来看看当执行任务队列的时候会发生什么，这段代码我们已经提过很多次了，算是注入器实现中的核心代码：

```js
// 执行模块中的任务队列
runInvokeQueue(moduleFn._invokeQueue);

// 对照定义constant的参数：(['$provide', 'constant', arguments])
// invokeArgs[0]: '$provide'
// invokeArgs[1]: 'constant':
// invokeArgs[2]: 类数组对象arguments
function runInvokeQueue(queue) {
  var i, ii;
  // 以此从任务队列中拿到任务，然后拿到对应的provider并进行调用
  for (i = 0, ii = queue.length; i < ii; i++) {
    var invokeArgs = queue[i],
        provider = providerInjector.get(invokeArgs[0]);

    provider[invokeArgs[1]].apply(provider, invokeArgs[2]);
  }
}
```

将定义`constant`所使用到的参数代入到上述for循环中，可以得到这么一段：

```js
// 假设我们定义了一个constant如下所示：
module.constant('a', 'aConstant');

// 代入到任务执行阶段：
var invokeArgs = ['$provide', 'constant', ['a', 'aConstant']],
      provider = providerInjector.get('$provide');

    provider['constant'].apply(provider, ['a', 'aConstant']);
```

注意其中的`['a', 'aConstant']`并不是一个真正的数组对象，它是一个由`arguments`所代表的类数组对象(Array-like Object)，关于类数组对象的定义，可以参考MDN对于它的[定义](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Functions/arguments)。

因此从这里我们就可以很明确地发现，负责提供`constant`这一菜谱的正是`$provide.constant`方法。也就是说，我们对`constant`的定义最后会被`$provide.constant('a', 'aConstant')`所落实。那让我们看看这一方法又做了些什么工作：

```js
// $provide直接被定义到了provider注入器的缓存中
providerCache = {
  $provide: {
      constant: supportObject(constant),
      // ......
    }
}

function supportObject(delegate) {
  return function(key, value) {
    if (isObject(key)) {
      forEach(key, reverseParams(delegate));
    } else {
      return delegate(key, value);
    }
  };
}

function constant(name, value) {
  // 确保常量的名字不叫做'hasOwnProperty'
  assertNotHasOwnProperty(name, 'constant');

  // 将常量直接定义到provider注入器和instance注入器的缓存中
  providerCache[name] = value;
  instanceCache[name] = value;
}

function assertNotHasOwnProperty(name, context) {
  if (name === 'hasOwnProperty') {
    // 抛出badname异常
    throw ngMinErr('badname', "hasOwnProperty is not a valid {0} name", context);
  }
}
```

可以发现，`$provide.constant`方法又被一个名为`supportObject`的函数给包装了一下。框架就是这样的，为了增加代码的复用性，总是会将一个函数层层包装，来达到最终的目的。其实这个`supportObject`函数做的事情也很简单，只是对`key`参数为对象的情况进行特殊处理。这个我们暂且不深究，等遇到了再作分析不迟。如果`key`不是对象，那么就直接交给`delegate`进行处理。放到`constant`这个上下文中，`delete`就是上述代码中的`constant`函数。

`constant`函数也只有寥寥三行代码。第一行确保`constant`的名字不为`hasOwnProperty`，因为`hasOwnProperty`本身就是JavaScript所有对象都拥有的一个方法，通过该方法可以判断一个对象是否拥有某个字段或者方法。所以不能将`constant`命名为这个名字，否则有可能会覆盖对象中的同名方法导致程序出现异常。

第二行和第三行做的事情就更简单直白了。将`constant`对应的`value`直接放入到`provider`注入器和instance注入器的缓存中。因此不仅在诸如`factory`，`service`等服务中我们可以声明`constant`依赖，在`provider`的构造器声明方式中也能够声明`constant`作为依赖，前文在介绍`provider`的时候讨论过如下一段代码：

```js
function provider(name, provider_) {
  assertNotHasOwnProperty(name, 'service');
  if (isFunction(provider_) || isArray(provider_)) {
    // provider的实例化由provider注入器完成，并非由instance注入器完成
    provider_ = providerInjector.instantiate(provider_);
  }
  if (!provider_.$get) {
    throw $injectorMinErr('pget', "Provider '{0}' must define $get factory method.", name);
  }
  return providerCache[name + providerSuffix] = provider_;
}
```

从代码中可以得知，`provider`的实例化由provider注入器完成，并非由instance注入器完成。所以在`provider`的构造函数中声明的依赖只能是保存在provider注入器的缓存中存在的依赖。也就是说，`provider`可以依赖于provider注入器缓存中的对象，也就是各种`provider`以及`constant`，但不能依赖其它存在于实例注入器缓存中的`factory`，`service`等等。毕竟二者的抽象层次不再一个级别上。`provider`作为`factory`，`service`等服务的生产者，可以看作是它们的"长辈"，"长辈"之间可以依赖，但是"长辈"不可以依赖它们的"晚辈"，是不是很有"长辈"的范。

所以，`constant`虽然简单，但是也有其特殊性，即两个注入器都会保留一份常量的实例，下面两种注入方式都是可行的：

```js
// 在provider的构造器函数中直接声明常量依赖
module.provider('b', function BProvider(a) {
  this.$get = function() {
    return 'constant: ' + a;
  };
});

// 在service中声明常量依赖
module.service('aService', function(a) {
  // ......
});

// 定义在最后也没关系：别忘了常量任务会通过unshift操作放到任务队列的头部
module.constant('a', 'aConstant');
```

通过上面的分析，想必对依赖注入和angular注入器内部的实现方式有更深入的了解了吧。在后续的文章中，会继续分析定义在module中的各种我们在开发angular应用时经常会使用到的方法。

