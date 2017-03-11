---
title: '[AngularJS面面观] 14. 依赖注入 --- module的定义与实现'
date: 2016-07-31 21:46:00
categories: [源码分析, AngularJS]
tags: [源码分析, AngularJS, 依赖管理]
---

从本篇文章开始，会开始系统性地介绍angular是如何实现依赖注入这一重要特性的。

## 引言

提到依赖注入，有后端背景的开发人员应该不会陌生。比如对于Java开发人员而言，绝大部分都是通过Spring这一框架首先了解到依赖注入这一概念的。所谓依赖注入(Dependency Injection)，它其实是一个更大的名为控制反转(Inverse of Control)概念的一种实现模式。只不过这种实现策略使用地太广泛了，导致出镜率非常高从而让很多人都觉得两者就是一回事。实际上，它们的关系是这样的：

![这里写图片描述](http://img.blog.csdn.net/20160720212635469)

<!-- More -->

控制反转思想更具有一般性和抽象性，感兴趣的同学可以参考另一篇文章[控制反转IoC概念随想](http://blog.csdn.net/dm_vincent/article/details/51972207)。

就依赖注入，一言以蔽之，它的作用是让框架帮你处理重要对象的生命周期的管理，不需要你显式地进行管理(对象构造和销毁)。这样能够让开发人员能够专注于应用的业务部分。

## angular如何实现模块

在谈angular是如何实现依赖注入之前，先看看angular是如何定义与实现模块的。毕竟，一个应用总是离不开模块的，应用使用到的各种服务，如controller，service，factory，provider等等，全部都是需要被定义在一个模块中的。

和angular模块相关的源代码全部保存在`loader.js`中。不到400行代码，注释占据了一大半。因此angular中用于实现模块的代码其实是非常简练的，让我们看看它是如何实现的。

### angular全局对象以及ensure方法

稍微深入一点用过angular的同学们都会知道，框架提供了一个名为`angular`的全局对象，在其之上定义了一些工具方法，DOM元素创建方法`element`，以及本文的重点`angular.module`方法。

那么这个全局对象是如何被创建出来的呢？

```js
function setupModuleLoader(window) {

  var $injectorMinErr = minErr('$injector');
  var ngMinErr = minErr('ng');

  function ensure(obj, name, factory) {
    return obj[name] || (obj[name] = factory());
  }

  var angular = ensure(window, 'angular', Object);

  // 将minErr服务添加到angular全局对象上
  angular.$$minErr = angular.$$minErr || minErr;

  return ensure(angular, 'module', function() {

    // 模块的创建方法...

  });
}
```

可以发现，angular全局对象通过调用`ensure(window, 'angular', Object)`得到。
而这个方法的定义也十分简单，更像是一些基本JavaScript操作的练习。`window`上已经存在了一个名为angular的对象，那么就直接返回。如果不存在则调用传入的factory方法进行构造并返回。因此，通过ensure方法得到的对象能够保证全局唯一性。

另外一些细节，比如前面用于声明异常如何处理的两行：

```js
var $injectorMinErr = minErr('$injector');
var ngMinErr = minErr('ng');
```

关于`minErr`，它是angular中提供的一个用于异常处理的服务。具体可以参看[这篇文章](http://blog.csdn.net/dm_vincent/article/details/51885375)，进行了详细介绍。

这里定义了两个异常包装类型，`$injectorMinErr`和`ngMinErr`。前者用于包装一些和`injector`相关的异常。关于`injector`，是实现依赖注入的关键，会在下篇文章中开始介绍。后者用于包装一些angular的基础异常。

### 模块注册和获取

至于angular.module方法，也是通过ensure方法来保证其在angular对象上的全局唯一性：

```js
return ensure(angular, 'module', function() {
  // 用来保存所有的模块
  var modules = {};

  return function module(name, requires, configFn) {
    var assertNotHasOwnProperty = function(name, context) {
      if (name === 'hasOwnProperty') {
        throw ngMinErr('badname', 'hasOwnProperty is not a valid {0} name', context);
      }
    };

    assertNotHasOwnProperty(name, 'module');
    if (requires && modules.hasOwnProperty(name)) {
      modules[name] = null;
    }
    return ensure(modules, name, function() {
      
      var moduleInstance = {
        requires: requires,
        name: name,
        // 模块对象提供的方法和对象定义在这里...
      };

      return moduleInstance;

      // ...
    });
  };
  
});
```

以上代码是setupModuleLoader函数的返回部分，清晰地表明了同样通过使用ensure函数，来保证了module函数在angular对象上的全局唯一性。

而其内部的`function module(name, requires, configFn)`就开始定义module函数本身的行为了。看到参数名称是不是很熟系了。第一个参数`name`用来定义模块的名称，二个参数`requires`是一个数组对象，用来定义该模块以来的模块名称。这个知识点只要是angular的入门文章或者书籍都会介绍吧。

这部分代码不停地在嵌套返回貌似很复杂的东西，比如上面的`return function module(name, requires, configFn)`，而这个函数内部又返回了一个：`ensure(modules, name, function(){ //... })`。看样子是怪吓人的，但是把握一条原则，调用ensure方法得到的只是一个对象，因此`ensure(modules, name, function(){ //... })`返回的只不过是`modules`对象的上的一个名为`name`的对象。而`modules`对象，其实就是用来保存所有模块的一个字典对象，模块的名字`name`作为其键值。所以很明显的，它返回的就是注册后得到的对象，典型的比如我们在初始化一个angular应用时所做的：

```js
var app = angular.module('testApp', []);
```

得到的app对象即为我们定义的模块，也就是`ensure(modules, name, function(){ //... })`的返回值，相信没有同学没写过上述代码吧。清楚了这个脉络，开始看看一些实现细节：

```js
var assertNotHasOwnProperty = function(name, context) {
  if (name === 'hasOwnProperty') {
    throw ngMinErr('badname', 'hasOwnProperty is not a valid {0} name', context);
  }
};

assertNotHasOwnProperty(name, 'module');
if (requires && modules.hasOwnProperty(name)) {
  modules[name] = null;
}
```

首先会保证`hasOwnProperty`这个名字是一个不合法的module名称。如果真碰到有不怀好意的开发者使用`hasOwnProperty`作为模块名称，那么angular直接就通过`ngMinErr`抛出异常宣布罢工不干了。

然后，就是通过判断是否传入了`requires`这个数组来决定是正在新建模块呢，还是正在获取已经存在的模块。如果存在`requires`，那么直接将当前可能存在的另外一个同名的模块置为空。这一个细节需要引起我们的注意。有些初学者没有分清楚注册模块和获取模块的区别，在获取模块的时候也传入了`requires`参数，这会导致不断地注册新的模块，而在原来的模块上定义的`controller`等等服务就都会随着这个置空操作而灰飞烟灭。所以就会出现莫名其妙的各种"未定义"异常。

最后，就是返回模块。如果是注册模块，就返回新注册的模块；如果是获取模块，就返回获取到的模块。这一切都是通过早先介绍的ensure函数完成的：

```js
return ensure(modules, name, function() {
      
  var moduleInstance = {
    requires: requires,
    name: name,
    // 模块对象提供的方法定义在这里...
  };

  return moduleInstance;

  // ...
});
```

`moduleInstance`就代表了这个模块，它将模块的名称以及依赖直接通过`name`和`requires`定义在其之上。当然，出了这两个最最基础的属性，`moduleInstance`还包含了很多东西，我们日常使用的诸如`controller`，`service`，`factory`，`constant`等方法都定义在它之上。它们的实现方式是我们即将讨论的内容。

至此，angular中和模块相关的操作就讨论完毕了。它作为后续即将介绍的angular核心功能之一依赖注入的一个铺垫。
