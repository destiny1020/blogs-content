---
title: '[AngularJS面面观] 15. 依赖注入 --- 初识注入器(Injector)'
date: 2016-08-05 00:23:00
categories: [源码分析, AngularJS]
tags: [源码分析, AngularJS, 依赖管理]
---

本篇文章继续介绍angular中实现依赖注入的"幕后英雄" --- 注入器(Injector)。说它是"幕后英雄"，是因为它才是依赖注入得以实现的主力。而上篇文章介绍的模块只不过是活跃在前台跟各位开发人员直接打交道的"接待人员"。

## 初识注入器

### 加载模块

在[上一篇文章](http://blog.csdn.net/dm_vincent/article/details/51884464)中，介绍了angular是如何定义和使用模块的。在这些模块中定义了各种应用需要的服务，注意模块只是定义了服务，而并没有真正地创建它们。真正的创建是通过注入器来完成的，那么怎么注入器是怎么知道需要创建哪些服务的呢？

<!-- More -->

我们都知道angular中可以定义常量，而定义的方法也很简单：

```js
var app = angular.module('test', []);
app.constant('testConstant', 'This is a constant value');
```

这里调用了`module`上的`constant`方法来**定义**这个创建常量的任务。注意"定义"这两个字，实际上在执行完这行代码后，名为`testConstant`的常量并没有被创建出来，只是定义了有这个任务存在。而真正的创建工作是在注入器介入后才会完成。

那么注入器是什么时候才会介入进来呢？答案就是，当模块被加载到注入器的时候，注入器就会知道被加载的模块中定义了哪些任务，从而介入到这个创建相应服务的任务中来了。

这也是创建注入器的入口函数中`modulesToLoad`这一参数的意义：

```js
function createInjector(modulesToLoad) {

  // HashMap是定义在apis.js中的一个类型，用来将数组转换为key具有一定生成规则的字典对象
  var loadedModules = new HashMap([], true);

  // 加载模块
  var runBlocks = loadModules(modulesToLoad);

  // 执行真正的创建工作
  forEach(runBlocks, function(fn) { if (fn) instanceInjector.invoke(fn); });

}
```

上面的`createInjector`函数只是真正实现的一小部分，但是为了循序渐进地讨论注入器的实现原理，这里就不忙将一些暂时无关的代码贴出来了，这样只会让大家更迷惑。下面的代码依然这样处理，只看和当前我们要讨论的内容密切相关的部分。

上述代码中重要的逻辑是模块在被加载后会发生什么，它由`loadModules`函数进行封装：

```js
 function loadModules(modulesToLoad) {
  var runBlocks = [], moduleFn;
  // 依次加载定义在依赖数组中的每一个module
  forEach(modulesToLoad, function(module) {
    // 如果已经加载过参数指定的module，立刻返回
    if (loadedModules.get(module)) return;
    loadedModules.put(module, true);

    if (isString(module)) {
      // 如果是以字符串形式定义的module --- 即module的名字
      moduleFn = angularModule(module);
      // 递归进行module的加载工作，因为被依赖的module同样还可以依赖于别的module(这里出现的runBlocks我们先行忽略，重点在其中loadModules函数的再次调用)
      runBlocks = runBlocks.concat(loadModules(moduleFn.requires)).concat(moduleFn._runBlocks);
      runInvokeQueue(moduleFn._invokeQueue);
    } else if (isFunction(module)) {
        // 如果是以函数形式定义的module
    } else if (isArray(module)) {
        // 如果是以数组形式定义的module
    } else {
      // 当module不符合上述几种类型时，抛出异常
      assertArgFn(module, 'module');
    }
  });
  
  return runBlocks;
}
```

有几个地方值得注意：

首先，加载模块并不是一个一蹴而就的过程。被依赖的模块也有可能依赖与别的模块，当发现当前正在加载的模块如果也依赖了别的模块，那么就会递归地进行加载，从而形成一个深度优先遍历的模块加载算法。
其次，定义依赖模块的方法可不止我们最常见的基于字符串的形式(也就是模块的名字)，从源代码的角度来看，还可以是函数或者数组。如若是这三种类型以外的，就会触发异常的抛出了。
最后，加载模块后得到的一个`runBlocks`其实就是我们前面所讨论的用于容纳注入器需要执行的任务的数组。

另外，上述方法中还用到了两个内部函数：
1. angularModule，它的定义在AngularPublic.js文件中。这个文件中定义的即是angular框架暴露给外部使用的所有公共服务和工具的定义。`angularModule = setupModuleLoader(window);`所以它是调用loader.js中用于初始化模块相关功能后的结果，也就是angular全局对象上的module属性。那么`angularModule(module)`的意思就很明显了，根据module的名字拿到对应的实例。
2. `assertArgFn`，该函数用于确保参数属于函数类型，如果参数类型错误则会抛出异常，与此类似的还有`assertArg`函数。它们都定义在Angular.js中。

下面我们就来具体看看任务队列究竟是啥。

### 任务队列

前面我们提到过在模块中，即使你定义了一个constant，这个constant也不会立马被创建出来。创建的只是一个等待注入器来执行的任务，所以就定义constant而言，看看具体的实现(由于实现在module中，因此代码在loader.js中)：

```js
// 一个module的实例包含的字段：现在又来了两个概念：_invokeQueue(任务队列)以及constant方法
var moduleInstance = {
  _invokeQueue: invokeQueue,  // invokeQueue会被初始化为一个空数组
  requires: requires,
  name: name,
  constant: invokeLater('$provide', 'constant', 'unshift')
}

// invokeLater函数的实现
function invokeLater(provider, method, insertMethod, queue) {
  if (!queue) queue = invokeQueue;
  return function() {
    queue[insertMethod || 'push']([provider, method, arguments]);
    return moduleInstance;
  };
}
```

constant被定义为调用`invokeLater('$provide', 'constant', 'unshift')`返回得到的一个函数。
那么我们顺着这个思路走一下，看看在调用诸如`app.constant('testConstant', 'This is a constant value');`时会发生什么。

做一些简单参数替换工作，执行的代码如下：

```js
invokeQueue['unshift'](['$provide', 'constant', 'testConstant', 'This is a constant value']);
return moduleInstance;
```

也就是把数组`['$provide', 'constant', 'testConstant', 'This is a constant value']`通过`unshift`方法置入到了`invokeQueue`数组的最前面。

那么这个`invokeQueue`数组中的数据在什么时候会被用到呢，前面在介绍`loadModules`函数的时候，如果传入的模块名的话，不是会执行下面这段代码嘛：

```js
if (isString(module)) {
  // 如果是以字符串形式定义的module --- 即module的名字
  moduleFn = angularModule(module);
  // 开始执行任务队列
  runInvokeQueue(moduleFn._invokeQueue);
} 
```

对，就是这个`runInvokeQueue`函数会触发任务队列的执行。也就是说，在加载模块的时候，定义在任务队列中的任务就会被执行。

它的实现如下所示：

```js
function runInvokeQueue(queue) {
  var i, ii;
  for (i = 0, ii = queue.length; i < ii; i++) {
    var invokeArgs = queue[i],
        provider = providerInjector.get(invokeArgs[0]);

    provider[invokeArgs[1]].apply(provider, invokeArgs[2]);
  }
}
```

这段代码会挨个地从任务队列中拿出数据，然后执行。执行的主体是一个叫做`provider`的对象，这个对象是通过调用一个名为`providerInjector`的东东的`get('$provide')`拿到的。最后调用provider上的`constant`方法，传入我们定义的`'testConstant', 'This is a constant value'`作为参数，完成了`constant`的真正创建工作。这里又出现了几个新玩意，`providerInjector`是个啥？这里涉及到了更多angular的实现原理和实现细节，我们暂且就不继续深入了，否则一定会晕掉！以后我们专门来讲讲`$provide`。

怎么样，以上就是注入器中关于模块加载以及任务执行的部分实现。新出现的概念比较多，如果有不明白的地方请慢慢回味。如果对模块这个概念也不熟悉的话，可以回顾上一篇文章中关于[模块](http://blog.csdn.net/dm_vincent/article/details/51884464)的介绍。

在下一篇文章中，我们会继续探讨注入器是如何真正实现依赖注入的。
