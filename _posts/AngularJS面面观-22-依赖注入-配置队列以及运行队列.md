---
title: '[AngularJS面面观] 22. 依赖注入 --- 配置队列以及运行队列'
date: 2016-08-21 14:14:00
categories: [源码分析, AngularJS]
tags: [源码分析, AngularJS, 依赖管理]
---

在上一篇文章中，介绍了`constant`的生命周期：它是如何被定义的，如何被创建，如何被使用的。本文继续介绍`module`上更多高层API的实现细节。在继续阅读下面的内容之前，还是建议对依赖注入本身要有足够的理解，当然如果你是跟着依赖注入的这一系列文章一路走来，对angular实现依赖注入的方式和细节应该是比较熟悉了。

本文会介绍定义与`module`上的两个方法：`module.config`以及`module.run`。表面上看它们好像没有什么区别，一个是配置另一个是运行。实际上，它们的区别还是挺大的。相信开发angular应用的同学们绝对不可能没用过`module.config`方法吧，因此我们就从它开始讨论。

<!-- More -->

### 配置队列(Config Queue)

首先，我们来思考思考配置队列这一机制的必要性。现在，我们已经知道依赖注入实际上是由两个注入器协力完成的。我们经常使用的是实例注入器，但是当实例注入器找不到某个对象的时候，还是会去provider注入器那里需求帮助。所以provider注入器目前就是一个被动的角色，平常无人问津，有问题了就会有实例注入器来找它帮忙。如果我们需要直接使用provider注入器中管理的对象，也不是没有办法，可以通过创建一个`provider`来完成，因为在`provider`的构造函数中是可以直接将provider注入器中管理的对象注入进来的。但是这样子是不是太麻烦了呢？为了使用其它`providers`，还需要单独创建一个毫无意义，只为使用它们的`provider`？这样真的好吗？而且我们知道，`provider`存在的意义之一也是为了配置，配置应该如何创建出真正的对象。所以应该提供一个地方能够无拘无束地使用`providers`来进行各种配置工作。在angular中，这个地方就是`module`上的`config`方法和它所对应的配置队列。

下面我们就来看看相关代码：

```js
var configBlocks = [];
var config = invokeLater('$injector', 'invoke', 'push', configBlocks);
moduleInstance.config = config;

// invokeLater函数的定义
function invokeLater(provider, method, insertMethod, queue) {
  if (!queue) queue = invokeQueue;
  // 还是利用柯里化将多个参数的函数转换为少数参数的函数
  return function() {
    // arguments才是我们在声明constant时实际传入的参数
    queue[insertMethod || 'push']([provider, method, arguments]);
    return moduleInstance;
  };
}
```

以上就是配置队列在`module`中的相关定义。这里仍然用到了我们的老朋友`invokeLater`函数。只不过最后显式地将`configBlocks`队列当作`queue`传入到了`invokeLater`函数中。因此，做一些基本的参数替换后，`config`方法的实际行为如下：

```js
var config = function() {
  configBlocks['push']('$injector', 'invoke', arguments);
};
```

那么`configBlocks`队列又是在何时被使用的呢？答案还是在注入器加载模块的那段代码中：

```js
if (isString(module)) {
  // 和运行队列相关的代码，马上就会介绍
  moduleFn = angularModule(module);
  runBlocks = runBlocks.concat(loadModules(moduleFn.requires)).concat(moduleFn._runBlocks);

  // 执行任务队列 - 这个我们已经相当熟悉了
  runInvokeQueue(moduleFn._invokeQueue);

  // 执行配置队列 - 这个是我们当前重点分析对象
  runInvokeQueue(moduleFn._configBlocks);
}
```

有几个细节需要把握，在执行配置队列中定义的配置任务之前，会执行任务队列。它的目的是保证在进行任何配置之前，将定义的各种服务都注册好。所谓注册，也就是将各种服务的底层`provider`都准备好。毕竟在配置任务中也是有可能需要使用这些服务对应的`provider`的，比如下面这段代码但凡用过angular的开发者都写过：

```js
// 如果你用Angular UI Router比较多，那么下面的$routeProvider就是$urlRouterProvider
module.config(function('$routeProvider') {
  // ......
});
```

注意在`config`方法接受的函数中被注入的参数是一个`provider`对象。这也是`config`方法的初衷，为了提供一个使用`providers`进行配置的地方。

那么具体而言，这个行为是如何实现的呢。其实就是把对于`config`函数的调用交给了`$injector.invoke`方法。关键就是这个`$injector`到底表示的是实例注入器呢，还是provider注入器呢：

```js
// 对照config的定义：(['$injector', 'invoke', arguments])
// invokeArgs[0]: '$injector'
// invokeArgs[1]: 'invoke':
// invokeArgs[2]: 类数组对象arguments
function runInvokeQueue(queue) {
  var i, ii;
  for (i = 0, ii = queue.length; i < ii; i++) {
    var invokeArgs = queue[i],
        provider = providerInjector.get(invokeArgs[0]);

    provider[invokeArgs[1]].apply(provider, invokeArgs[2]);
  }
}
```

注意看for循环中的这一行代码：

```js
// 相当于调用的providerInjector.get('$injector')
provider = providerInjector.get(invokeArgs[0]);
```

所以是期望从provider注入器中拿到`$injector`，而provider注入器中的`cache`有没有保存这个`$injector`对象呢？答案是保存了，而且`$injector`指的就是provider注入器自己：

```js
providerInjector = (providerCache.$injector =
          createInternalInjector(providerCache, function(serviceName, caller) {
            if (angular.isString(caller)) {
              path.push(caller);
            }
            throw $injectorMinErr('unpr', "Unknown provider: {0}", path.join(' <- '));
          })),
```

创建provider注入器之后将它保存在了自身的缓存中。因此对于`config`中定义的函数会被provider注入器进行调用，所以`config`函数中定义的依赖全部都来源于provider注入器的缓存。这就是整个`config`从定义到执行的过程，其实也没什么特别的，关键的概念我们之前几乎都已经接触过了，有的还不止一次。只要了解了注入器的工作原理可谓是轻车熟路。

除了通过调用`module.config`来注册一段配置任务之外，angular其实还提供了另一种方法。就是在声明`module`时传入的第三个参数：

```js
// 第三个参数configFn就是需要执行的配置函数
function module(name, requires, configFn) {}

// 在内部同样还是通过调用config方法完成配置函数的注册
if (configFn) {
  config(configFn);
}
```

使用哪一种方式进行配置函数的注册完全是看开发人员的个人爱好。但是我觉得还是使用`module.config`来注册配置函数更好，代码的可读性似乎更高一些。

### 运行队列(Run Queue)

在`module`中定义了另外一个名为`run`的方法。从文档上来看：

```js
/**
 * @ngdoc method
 * @name angular.Module#run
 * @module ng
 * @param {Function} 在注入器被创建后执行。用于应用的初始化。
 * @description
 *  使用此方法来注册那些注入器加载完成所有模块后需要执行的任务。
 */
run: function(block) {
  runBlocks.push(block);
  return this;
}
```

好像和`config`没什么太的区别？都是运行一段代码。还是看看在创建注入器时和运行队列相关的行为：

```js
// createInjector函数中和运行队列相关的代码
// loadModules加载模块的函数返回值就是运行队列
var runBlocks = loadModules(modulesToLoad);
instanceInjector = protoInstanceInjector.get('$injector');
// 依次运行每个运行任务
forEach(runBlocks, function(fn) { if (fn) instanceInjector.invoke(fn); });
```

这里有三个值得注意的细节：
第一个是`loadModules`函数其实是有返回值的，这个返回值就是运行队列。
第二个是每个执行队列中的任务都是通过实例注入器来调用的： `instanceInjector.invoke(fn)`。这一点和前面介绍的配置队列也不太一样，配置队列是通过provider注入器调用。
第三个是运行队列的执行是在加载完**所有**模块之后才发生的，这一点和配置队列以及前面介绍的任务队列不太一样，后两者是在加载某个模块时就会发生的。既然运行队列的执行是在加载完**所有**模块之后才发生的，那么就需要将各个模块中定义的运行队列都收集在一起，然后才能统一执行。那么又是在什么地方将这些运行任务都收集起来的呢？答案就在loadModules函数的实现中：

```js
function loadModules(modulesToLoad) {
  var runBlocks = [];
  forEach(modulesToLoad, function(module) {
    // ......

    try {
      if (isString(module)) {
        moduleFn = angularModule(module);
        // module的递归加载在这里发生，运行队列的收集也在这里进行
        runBlocks = runBlocks.concat(loadModules(moduleFn.requires)).concat(moduleFn._runBlocks);

        // 执行当前正在被加载模块的任务队列以及配置队列
        runInvokeQueue(moduleFn._invokeQueue);
        runInvokeQueue(moduleFn._configBlocks);
      } else if (isFunction(module)) {
        // 如果模块是一个函数，那么使用provider注入器调用之，并将其返回值放入到执行队列
        runBlocks.push(providerInjector.invoke(module));
      } else if (isArray(module)) {
        // 如果模块是一个数组对象，那么使用provider注入器调用之，并将其返回值放入到执行队列
        runBlocks.push(providerInjector.invoke(module));
      } else {
        assertArgFn(module, 'module');
      }
    } catch (e) {
      // ......
    }
  });
  return runBlocks;
}
```

可见，执行队列的收集工作是伴随着模块的递归加载而完成的。在加载一个模块的时候，只会对执行队列进行收集，而不像之前介绍的任务队列和配置队列那样，加载模块的同时就会调用`runInvokeQueue`函数来实际地执行它们。

同时，我们也发现了`module`除了是字符串类型(也就是模块的名称)之外，还能够是一个函数类型。当被声明为函数类型时，会有单独的处理：使用provider注入器调用，并将其返回值放入到执行队列。这又是在玩哪一出呢？`module`居然还可以无名无姓，就是一个函数？其实，这种直接以函数的形式定义的`module`的作用就相当于是一个定义了配置任务和执行任务的综合体。

来分析比较一下下面两段代码便知：

```js
// 这是定义在module中的config方法，本质上它还是调用的provider注入器的invoke方法
var config = invokeLater('$injector', 'invoke', 'push', configBlocks);

// 这是在将module定义为函数时的调用方式，直白地使用了provider注入器的invoke方法
runBlocks.push(providerInjector.invoke(module));
```

它们两者的定义方式虽然不同，但是都有一颗`providerInjector.invoke`的心脏。所以在这样定义的`module`中，也是可以将各种`providers`和`constant`作为依赖声明为参数的，比如这样：

```js
// 假设'a'是一个常量，'bProvider'是一个provider
angular.module(function(a, bProvider){});
```

这种方式提供了一个方便快捷地定义配置任务的方案，如果只是希望进行简单的配置工作，完全可以不通过传统的`module.config`来进行任务的注册。直接使用函数形式的`module`即可。而且，为了让它更加方便，它的返回值也被利用上了，返回值函数会被当作执行任务投放到`runBlocks`中，因此下面这种定义方式直接定义了一个`config`任务和一个`run`任务：

```js
// 假设'a'是一个常量，'bProvider'是一个provider, 'cService'是一个service
angular.module(function(a, bProvider){
  // 一些配置工作
  return function(a, cService) {
    // 一些初始化工作
  };
});
```

### 和依赖注入相关的三种队列

至此，我们已经接触到了和依赖注入密切相关的三种队列，下面简单总结回顾一下它们的区别：

#### 任务队列(Invoke Queue)

调用`module`上的高层API就会向任务队列中增加一个相应任务，比如`module.constant`的调用就导致了一个常量任务的创建，具体而言就是定义在依赖注入模块(injector.js)中的`constant`函数的执行：

```js
function constant(name, value) {
  assertNotHasOwnProperty(name, 'constant');
  providerCache[name] = value;
  instanceCache[name] = value;
}
```

除此之外，还有对应于`config`中`factory`，`service`等方法的定义在注入器中的`factory`函数和`service`函数等。对于`factory`，`service`等任务而言，它们的执行并不会导致真正的对象被创建，而是注册一个具体的`provider`，这个`provider`知道该如何创建真正的对象，待需要的时候创建之，从而实现"懒加载"。执行时机上，任务队列的执行发生在模块被加载的时候。

#### 配置队列(Config Queue)

本文介绍的配置队列是为了提供一个配置各种providers的地方，通过provider注入器完成调用。任务队列和执行队列中定义的任务都是通过实例注入器进行调用的，因此它们无法直接地注入定义在provider注入器中的各种providers。配置队列的执行时机和任务队列相似，都是在加载某个模块的时候就会被执行，但是在顺序上它的执行发生在同模块的任务队列执行之后。

#### 运行队列(Run Queue)

它用于定义一些模块的初始化工作。和任务队列一样，定义在运行队列中的任务都是通过实例注入器完成实际的调用工作的。但是和它以及配置队列不一样的是，它的执行时机实在**所有**模块全部加载完毕之后，此时所有的服务都已经完成注册工作(各路providers都已经准备好，知道如何初始化托管对象)。所以当运行队列中的任务被执行时，它是可以将需要的各种依赖都声明在其参数列表中的，哪怕这些依赖被定义在不同的模块中。

---

至此，我们已经了解到了和依赖注入实现息息相关的三种队列结构以及它们各自的特点和用法。在下一篇文章中会继续介绍`module`中剩下的几个方法，这些方法在我们的实际应用中会经常被用到，比如`factory`，`service`等。

