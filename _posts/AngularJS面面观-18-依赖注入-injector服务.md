---
title: '[AngularJS面面观] 18. 依赖注入 --- $injector服务'
date: 2016-08-09 09:24:00
categories: [源码分析, AngularJS]
tags: [源码分析, AngularJS, 依赖管理, 注解]
---

有了前面那么多的铺垫工作，`$injector`服务正式上线。本文将介绍angular提供给开发者可以直接使用的`$injector`服务中包含的可调用方法以及每个方法的实现。

## $injector服务

首先我们看看这个服务中包含了那些方法：

```js
return {
  invoke: invoke,
  instantiate: instantiate,
  get: getService,
  annotate: createInjector.$$annotate,
  has: function(name) {
    return providerCache.hasOwnProperty(name + providerSuffix) || cache.hasOwnProperty(name);
  }
};
```

<!-- More -->

### get以及has

先挑软柿子捏，看看`get`和`has`方法的实现。

`get`方法的实现如下：

```js
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

这个方法其实在前面的文章中已经介绍过了，这里再来回顾一下加深印象：
首先`$injector`的内部维护了一个`cache`字典对象用于保存被注入器托管的对象。由于`getService`除了真正获取需要的被托管对象外，还有一个职责就是当对象不存在时，尝试初始化该对象。而初始化对象的时候，必须要考虑到该对象很有可能也依赖了其它更多的对象，因此这个初始化的过程实际上是一个嵌套和递归的过程。随着层次的深入，有可能出现循环依赖的问题。所以将对象设置为`INSTANTIATING`的目的就是设置一个标志位，表示该对象正在实例化了，如果再次尝试实例化一个已经处于实例化状态的对象，就表示发生了循环依赖，需要抛出异常提示开发者，`path`数组就是为了记录实例化路径从而当异常发生的时候能够提醒开发者而存在的。最终，对象的实例化是通过调用`cache[serviceName] = factory(serviceName, caller)`来完成的。关于`factory`的具体用法我们留在后面介绍`provider`的时候再来分析，现在知道它才是真正负责实例化对象的就够了。

所以`get`方法不仅仅实现了"拿"操作，当要"拿"的对象并不存在的时候，还会顺带尝试实例化这个对象。

`has`方法的实现如下：

```js
has: function(name) {
  return providerCache.hasOwnProperty(name + providerSuffix) || cache.hasOwnProperty(name);
}
```

判断一个对象是否存在，会去两个地方找：
1. `providerCache`，键值为`name`加上一个特定的`providerSuffix`后缀，这个后缀的值为`Provider`。
2. `cache`，键值为`name`。

关于第一点，我们暂且还不知道`Provider`为何物，只是打过照面。因此这部分先不必理会，待介绍`Provider`的时候一切就水到渠成了。而第二点，则是我们已经熟知的`cache`字典对象，保存了一系列被注入器托管的对象。

### annotate

下面是获取注解信息的`annotate`方法了：

```js
function annotate(fn, strictDi, name) {
  var $inject,
      argDecl,
      last;

  if (typeof fn === 'function') {
    if (!($inject = fn.$inject)) {
      // 没有提供$inject并且非严格模式时，使用源码解析的方式构建$inject
      $inject = [];
      if (fn.length) {
        if (strictDi) {
          if (!isString(name) || !name) {
            name = fn.name || anonFn(fn);
          }
          throw $injectorMinErr('strictdi',
            '{0} is not using explicit annotation and cannot be invoked in strict mode', name);
        }
        argDecl = extractArgs(fn);
        forEach(argDecl[1].split(FN_ARG_SPLIT), function(arg) {
          arg.replace(FN_ARG, function(all, underscore, name) {
            $inject.push(name);
          });
        });
      }
      fn.$inject = $inject;
    }
  } else if (isArray(fn)) {
    // 当使用Array-Style的声明方式时，去掉最后一个元素即为$inject
    last = fn.length - 1;
    assertArgFn(fn[last], 'fn');
    $inject = fn.slice(0, last);
  } else {
    // 抛出异常
    assertArgFn(fn, 'fn', true);
  }

  // 得到注解信息供后续使用
  return $inject;
}
```

关于注解信息，在前文中已经重点分析过了。它的主要作用是提供形式参数到真正被托管对象的一个关联。在angular中有三种提供这种关联关系的注解方式：

1. 直接通过`$inject`数组指定注解
2. 提供数组风格的注解(它其实就是第一种基于`$inject`方式的一种快捷写法)
3. 基于源代码解析的方式(默认开启，通过`strictDi`控制)

更多关于注解的分析，可以参考这篇文章：[17. 依赖注入 --- 注解的定义与实现](http://blog.csdn.net/dm_vincent/article/details/52081180)

### invoke

`invoke`方法的实现如下：

```js
function invoke(fn, self, locals, serviceName) {
  // 如果提供了locals并且它是字符串类型，则用它替换serviceName
  if (typeof locals === 'string') {
    serviceName = locals;
    locals = null;
  }

  // 调用injectionArgs来完成真正参数的获取，实现在下面
  var args = injectionArgs(fn, locals, serviceName);
  if (isArray(fn)) {
    fn = fn[fn.length - 1];
  }

  // 判断fn是不是构造器函数，对待普通函数和构造器函数的处理方式不同(仅针对IE)
  if (!isClass(fn)) {
    // http://jsperf.com/angularjs-invoke-apply-vs-switch
    // #5388
    return fn.apply(self, args);
  } else {
    args.unshift(null);
    return new (Function.prototype.bind.apply(fn, args))();
  }
}

function injectionArgs(fn, locals, serviceName) {
  var args = [],
      // 获取注解信息
      $inject = createInjector.$$annotate(fn, strictDi, serviceName);

  for (var i = 0, length = $inject.length; i < length; i++) {
    var key = $inject[i];
    // 确保$inject中的每个key都是字符串类型，否则抛出异常
    if (typeof key !== 'string') {
      throw $injectorMinErr('itkn',
              'Incorrect injection token! Expected service name as string, got {0}', key);
    }
    // 通过注解的key来得到的真正依赖的对象 --- 如果locals中提供了拥有相同键值的对象，则优先使用它
    args.push(locals && locals.hasOwnProperty(key) ? locals[key] :
                                                     getService(key, serviceName));
  }
  return args;
}

// 从源码来看仅针对IE浏览器
function isClass(func) {
  // IE 9-11 do not support classes and IE9 leaks with the code below.
  if (msie <= 11) {
    return false;
  }
  // Workaround for MS Edge.
  // Check https://connect.microsoft.com/IE/Feedback/Details/2211653
  return typeof func === 'function'
    && /^(?:class\s|constructor\()/.test(Function.prototype.toString.call(func));
}
```

上述代码不少，但是主线逻辑很明确：
1. 通过`injectionArgs`来获取实际的参数，即获得被托管对象(通过前面介绍的`getService`方法)，如果传入的`locals`中提供了覆盖，则优先使用locals中定义的。
2. 判断`fn`是不是构造器函数，对待普通函数和构造器函数的处理方式不同(但是目前从源码来看仅针对IE)。所以如果是想通过注入器调用构造器函数的话，还是更推荐使用下面即将介绍的`instantiate`方法。`invoke`方法仅作为调用普通函数的方法使用。

`invoke`方法看似平淡无奇，我们也不怎么主动去调用它，但是它实则是注入器实现中的关节一环：

```js
protoInstanceInjector =
          createInternalInjector(instanceCache, function(serviceName, caller) {
            var provider = providerInjector.get(serviceName + providerSuffix, caller);
            return instanceInjector.invoke(
                provider.$get, provider, undefined, serviceName);
          }),
```

上述代码看不懂没关系，等介绍了`provider`我们再看来，现在贴出这段代码只是想让大家不要忽视了`invoke`方法，我们不怎么调用它并不代表它没有用。实际上它绝对是注入器的王牌方法，其它的方法或多或少都是为了它而存在的。

### instantiate

了解了`invoke`方法，再来看看`instantiate`方法就容易多了：

```js
function instantiate(Type, locals, serviceName) {
  // 当Type为数组时，根据基于数组风格的注解所规定的那样，数组中的最后一个元素才是函数
  // 比如. someModule.factory('greeter', ['$window', function(renamed$window) {}]);
  var ctor = (isArray(Type) ? Type[Type.length - 1] : Type);
  var args = injectionArgs(Type, locals, serviceName);
  // 处于第一个位置的空元素在使用new来调用函数的时候是必要的
  args.unshift(null);
  return new (Function.prototype.bind.apply(ctor, args))();
}
```

这两个方法是的区别主要在于调用传入`function`的方式不同。`invoke`就是普通的调用方式，而`instantiate`则是通过`new`来把函数当作构造器函数进行调用。当我们在`module`上定义`service`，`provider`的时候，实际上内部就是调用的`instantiate`函数来完成对象的创建。

---

因此，`invoke`和`instantiate`方法虽然在实际应用的开发过程中，直接使用的机会很少很少(当然，写单元测试的时候不会少接触)。但是它们的作用在整个angular框架的架构中实在是太重要了，依赖注入就是依赖它们才能够正常工作的，所以了解它们的实现思想和实现细节真的是非常有好处。

当然，还有好多细节我们现在还没法看懂，比如好多地方出现了`provide`或者是`provider`这种概念。另外，一些我们常用的`controller`，`service`，`factory`等等都是如何通过注入器来实现的也是亟待弄清楚的问题。别担心，从下一篇文章开始就会开始系统性地介绍它们。