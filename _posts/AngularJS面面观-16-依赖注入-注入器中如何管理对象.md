---
title: '[AngularJS面面观] 16. 依赖注入 --- 注入器中如何管理对象'
date: 2016-08-06 18:19:00
categories: [源码分析, AngularJS]
tags: [源码分析, AngularJS, 依赖管理]
---

上一篇文章初次介绍了注入器(Injector)，分析了它加载模块的过程以及它是如何执行任务队列的。这里需要重申一下的是，所谓任务队列实际上就是我们在开发一个基于angular的应用时定义的那些`constant`，`service`，`factory`等等，它们通过`module`类型提供的方法定义，但是定义并不代表立即就创建。它们的创建工作是交给注入器来完成的。

那么执行了这些任务后，会产生什么效果，这又和我们讨论的主题-依赖注入有什么关联呢？这就是我们在这篇文章中需要讨论的。

<!-- More -->

## 依赖注入总览

在继续讨论之前，我们需要看看依赖注入到底在angular应用中意味着什么。众所周知，在开发一个angular应用时，我们可以直接将需要的服务以参数的形式定义在所定义的各种angular提供的类型中，比如`controller`，`service`，`factory`等等，下面是一段典型代码：

```js
angular.module('test').controller('testController', function($rootScope, testConstant, testFactory) {
  // 利用$rootScope, testConstant, testFactory完成业务逻辑
}); 
```

在感受到这种业务代码编写方式便利性的时候，你有没有感觉到哪怕是一丝半毫的不可思议呢？我们需要的各种服务和数据为什么就能够以这种直接定义成参数的形式得以实现呢？

angular框架是如何得知上述参数中定义的`$rootScope`, `testConstant`, `testFactory`是从何而来呢？如果我们把它们换个名字，比如换成`$rootScope1`, `testConstant1`, `testFactory1`，还能不能得到相同的结果呢？

带着这些问题，我们来看看依赖注入是如何解决这个问题的。所谓依赖注入，实际上是控制反转(Inverse of Control)设计思想的一种具体模式。而控制反转的核心思想等同于所谓的"好莱坞原则"："不要打电话给我们，我们会打电话给你"。套用在依赖注入的上下文中，这句话就演绎成了："不要去主动寻找和创建需要的服务，给个名字交给注入器帮你搞定"。因此在这个前提条件下，才有了我们在前述代码中所看到的那样，我们在参数列表中声明了我们需要的服务和数据，注入器真的就帮我们搞定了。而且上述`controller`定义的function并不是由应用程序来调用，它是通过angular框架进行调用，这样才能够将真正的参数传入进去。从这个角度而言，它也实践了控制反转这一原则。作为应用程序开发者的你，只需要根据规范定义好业务逻辑即可，后续的一切工作全部交给框架处理。

弄明白了这个问题，剩下的问题就变成了：注入器是如何搞定的呢？我们只提供了一个名字，它就搞定了？感觉很神奇吧，其实我们在前面已经给出了这个问题的答案 --- 任务队列。

### 执行任务队列的目的

执行任务队列的目的有两点：

1. 为了创建这些定义的数据(比如简单的一点的`constant`，`value`等)。这类数据往往比较简单，即使直接创建出来也占用不了太多资源。
2. 为了提供给注入器如何创建服务的"蓝图"(比如复杂一点的如`controller`，`service`，`factory`等)，这样做的目的是为了实现"懒加载"，因为这类对象往往会比较复杂，如果在注入器在加载模块的时候就一股脑地将它们全部给创建出来了，然而应用中却没有使用它们，岂不是很亏？

### 注入器如何管理对象

然后在需要某个对象的时候，注入器会首先来看它是不是已经存在了。如果存在的话就直接返回，如果不存在就会先创建，然后保存到缓存中并返回。相关代码如下所示：

```js
// 所有被注入器管理的对象缓存
var instanceCache = {};

// 获取服务的函数，函数体中的cache就是上面的instanceCache
function getService(serviceName, caller) {
  // 检查缓存中是否已经存在需要的服务
  if (cache.hasOwnProperty(serviceName)) {
    // 如果发现该服务已经被标注为"正在实例化"，则抛出循环依赖异常
    if (cache[serviceName] === INSTANTIATING) {
      throw $injectorMinErr('cdep', 'Circular dependency found: {0}',
                serviceName + ' <- ' + path.join(' <- '));
    }
    return cache[serviceName];
  } else {
    try {
      // 将服务名置入到path数组中，记录实例化服务的顺序
      path.unshift(serviceName);
      // 将服务标注为"正在实例化"
      cache[serviceName] = INSTANTIATING;
      // 调用factory来实例化得到服务对象 --- 此时才真正得到了服务对象
      return cache[serviceName] = factory(serviceName, caller);
    } catch (err) {
      // 发生异常时，清除实例化出错的服务
      if (cache[serviceName] === INSTANTIATING) {
        delete cache[serviceName];
      }
      throw err;
    } finally {
      // 清除最后的路径信息
      path.shift();
    }
  }
}
```

如注释所解释的那样，以上的代码主要干了这么几件事情：
1. 判断是否发生循环依赖，如果发生了是要抛出异常的
2. 被注入器托管对象的管理 --- 创建和获取

---

#### 循环依赖的检测

对于第一点，循环依赖的问题。相信有一些实际angular开发经验的同学们一定已经遇到过了。下面这段代码重现了这个问题：
```js
<html ng-app="test">
<head>
	<title>Angular Circular Dependency Example</title>
</head>
<body ng-controller="testController">
	Test
</body>
<script src="//cdn.bootcss.com/angular.js/1.5.8/angular.js"></script>
<script type="text/javascript">
	var module = angular.module('test', []);

	module.service('service1', function(service2) {});
	module.service('service2', function(service1) {});

	module.controller('testController', function(service1) {});
</script>
</html>
```

上面定义的两个`service`互相依赖于对方。但是根据注入器"懒加载"的特性，如果仅仅定义了两个`service`而不定义在哪使用它们的话，也是不会触发注入器的实例化操作的。因此还定义了一个`controller`，并在body元素上声明了使用该控制器，用来触发注入器实例化的行为，进而触发我们所期待的循环依赖异常。

如果运行这个例子就会出现下面的异常：

```js
angular.js:13920 Error: [$injector:cdep] Circular dependency found: service1 <- service2 <- service1
http://errors.angularjs.org/1.5.8/$injector/cdep?p0=service1%20%3C-%20service2%20%3C-%20service1
    at angular.js:68
    at getService (angular.js:4656)
    at injectionArgs (angular.js:4688)
    at Object.instantiate (angular.js:4730)
    at Object.<anonymous> (angular.js:4573)
    at Object.invoke (angular.js:4718)
    at Object.enforcedReturnValue [as $get] (angular.js:4557)
    at Object.invoke (angular.js:4718)
    at angular.js:4517
    at getService (angular.js:4664)
```

值得一提的是，上面的示例程序中使用的是未经过压缩混淆的angular源代码。如果你使用的是压缩混淆过的angular.min.js。输出就不会这么详尽了：

```js
Error: [$injector:cdep] http://errors.angularjs.org/1.5.8/$injector/cdep?p0=service1%20%3C-%20service2%20%3C-%20service1
    at Error (native)
    at http://cdn.bootcss.com/angular.js/1.5.8/angular.min.js:6:412
    at d (http://cdn.bootcss.com/angular.js/1.5.8/angular.min.js:40:349)
    at e (http://cdn.bootcss.com/angular.js/1.5.8/angular.min.js:41:158)
    at Object.instantiate (http://cdn.bootcss.com/angular.js/1.5.8/angular.min.js:42:24)
    at Object.<anonymous> (http://cdn.bootcss.com/angular.js/1.5.8/angular.min.js:42:352)
    at Object.invoke (http://cdn.bootcss.com/angular.js/1.5.8/angular.min.js:41:456)
    at Object.$get (http://cdn.bootcss.com/angular.js/1.5.8/angular.min.js:39:142)
    at Object.invoke (http://cdn.bootcss.com/angular.js/1.5.8/angular.min.js:41:456)
    at http://cdn.bootcss.com/angular.js/1.5.8/angular.min.js:43:265
```

大家可以对比一下输出的不同。除了调用栈不同之外，前者还多了循环依赖的详细信息：

```
Circular dependency found: service1 <- service2 <- service1
```

而以上循环依赖的数据来源正是`path`数组。因此，在开发一个angular应用的时候，也建议大家使用angular.js，而非angular.min.js。因为在发生异常时前者能够提供更多的信息。关于angular异常的封装，可以参考我的另外一篇[文章](http://blog.csdn.net/dm_vincent/article/details/51885375)，对这个问题进行了探讨。

发生循环依赖的本质还是在于注入器在实例化服务对象的时候，采用的算法也是深度优先遍历，这一点在原理上和注入器处理模块的加载是别无二致的。因为一个服务对象也可能需要首先依赖更多的其它服务对象，这样逐层深入下去，就成了一张服务对象依赖关系图。这张单向图需要是一张有向无环图(DAG)，否则就无法解释到底是谁依赖谁了。因此，这也算是拓扑排序算法在angular中的一个简单应用吧。关于拓扑排序，有兴趣的同学还可以看[这篇文章](http://blog.csdn.net/dm_vincent/article/details/7714519)，欢迎大家来探讨。

---

#### 被托管对象的创建和获取

关于第二点，注入器对被托管对象的创建和获取。
就创建而言，分为简单对象和复杂对象。像诸如`constant`，`value`这样的简单对象，直接定义到缓存中即可。对于复杂对象，调用通过执行任务队列得到的"蓝图"即可，也就是`getService`方法中的`factory`函数。就获取而言，它和创建其实是相辅相成的，创建了之后才能获取。

## 结语

本篇文章介绍了依赖注入的原理，以及angular是如何实践这一原理的。当然，这还没完。现在介绍的只是angular的注入器是如何管理被托管对象的，离实际应用还差了一点料。这个料就是下一篇文章的主题 --- 注解的定义与实现。正是这个料最终促成了angular中依赖注入服务`$injector`的诞生。
