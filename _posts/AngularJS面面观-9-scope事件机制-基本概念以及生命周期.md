---
title: '[AngularJS面面观] 9. scope事件机制 - 基本概念以及生命周期'
date: 2016-07-04 00:09:00
categories: [源码分析, AngularJS]
tags: [源码分析, AngularJS]
---

在前面的8篇文章中，已经介绍了scope的几个方面，比如digest循环，继承机制等。
本篇文章以及后续的两篇文章打算把这块拼图完成，讨论一下scope的另外一项重要功能：事件机制。

主要分为以下6个方面进行讨论：
1. 发布-订阅模式(Publish-Subscribe Pattern)
2. 事件的生命周期-注册和注销
3. 事件与scope继承树-`$emit`以及`$broadcast`
4. 事件对象的组成
5. 事件的停止传播以及阻止默认行为
6. 事件在angular框架中的应用

本文首先讨论1-2。下一篇文章中讨论3-5。最后一篇文章讨论6。

<!-- More -->

## 发布-订阅模式(Publish-Subscribe Pattern)

这个模式算是设计模式中经典中的经典了。是消息中间件必然要实现的模式之一，也被众多框架以及API所支持。该模式在大规模分布式系统中是不可或缺的，它将系统中的消息产生者和消息消费者解耦开来，提高了系统的可扩展性。关于这个模式的概念和定义，这里就不再赘述了。对这个概念还不熟悉的同学们可以移步这里学习一下：[英文Wiki](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern)。想看具体代码的同学网上搜索一下相关资料非常多。

那么当上下文变成了angular，发布订阅模式又是如何被实现和应用的呢？
就实现而言，angular的发布订阅模式和其它的各种实现并没有什么本质上的区别，但angular事件系统最重要的不同在于：它已经和scope继承树融为一体了。这也是为什么事件机制会被实现在scope中的原因，毕竟就第一印象而言，谁也不会将事件机制和scope这种数据载体以及继承机制混为一谈。

那么问题来了，事件机制怎么和scope继承树融为一体的呢？嗯，不要心急，待我一一道来。
让我们先看看angular中事件的生命周期，事件是如何注册以及注销的。

## 事件的生命周期-注册

实现其实不复杂，直接贴上代码一起分析一下：

```js
$on: function(name, listener) {
  // listeners字典对象的初始化
  var namedListeners = this.$$listeners[name];
  if (!namedListeners) {
    this.$$listeners[name] = namedListeners = [];
  }
  namedListeners.push(listener);

  // 更新listenerCount计数器字典对象
  var current = this;
  do {
    if (!current.$$listenerCount[name]) {
      current.$$listenerCount[name] = 0;
    }
    current.$$listenerCount[name]++;
  } while ((current = current.$parent));

  // 提供注销该listener的函数
  var self = this;
  return function() {
    var indexOfListener = namedListeners.indexOf(listener);
    if (indexOfListener !== -1) {
      namedListeners[indexOfListener] = null;
      decrementListenerCount(self, 1, name);
    }
  };
}

// 内部使用的用于减少计数器的方法
function decrementListenerCount(current, count, name) {
  do {
    current.$$listenerCount[name] -= count;

    if (current.$$listenerCount[name] === 0) {
      delete current.$$listenerCount[name];
    }
  } while ((current = current.$parent));
}
```

首先我们根据文档来看看这个	`$on`方法的签名：

```
/**
/* @param {string} name 需要监听的事件名称。
 * @param {function(event, ...args)} 当事件发生时，需要执行的listener回调函数。
 * @returns {function()} 返回一个用户注销事件的函数。
 */ 
$on: function(name, listener){}
```

一目了然，这里在注册事件的时候就考虑到了注销，而采取的策略和在scope中定义watcher一致，都是返回一个注销函数，当需要注销此回调函数的时候调用一下即可。

`$on`的实现可以分为3个部分：
1. 初始化以及记录回调函数：初始化listeners字典对象并且将回调函数保存到其中。
2. 计数器更新：更新listenerCount计数器字典对象。
3. 返回注销函数：调用后更新listeners字典对象以及listenerCount计数器字典对象。

`$on`的实现基本没有什么比较晦涩难懂的地方，简洁明了。

至于一些细节部分，不懂的地方仔细体会一下即可。
而上面提到的两个字典对象：listeners和listenerCount，都是在创建子scope的时候就会创建的：
```js
function createChildScopeClass(parent) {
  function ChildScope() {
    this.$$watchers = this.$$nextSibling =
        this.$$childHead = this.$$childTail = null;
    this.$$listeners = {};  // 用于保存事件对应的回调函数
    this.$$listenerCount = {};  // 用于保存事件对应回调函数的计数器
    this.$$watchersCount = 0;
    this.$id = nextUid();
    this.$$ChildScope = null;
  }
  ChildScope.prototype = parent;
  return ChildScope;
}
```

## 事件的生命周期-注销

对于事件的注销，过程上和注册非常相似，只不过正好相反：
```js
return function() {
  var indexOfListener = namedListeners.indexOf(listener);
  if (indexOfListener !== -1) {
    namedListeners[indexOfListener] = null;
    decrementListenerCount(self, 1, name);
  }
};
```
在调用注销函数后，首先得到注册的回调函数在数组中的位置，当存在时将其置为空并更新计数器。

总结一下，目前发现的angular中事件的几个特点：
1. 事件保存在scope上，也就是说和scope一样，事件也是拥有一个树形结构的。嗯，这里就可以初步看出angular的事件机制是和scope融为一体的。
2. 除了事件对应的回调函数外，还记录了回调函数的个数。
3. 提供了和watcher注销机制类似的注销机制。

## angular事件和浏览器事件

一些题外话。

以上提到的angular事件和浏览器事件(比如，点击事件click，键盘输入事件keydown等)不是一回事。angular事件更多的应用内自定义事件。在使用时，事件的触发和事件的响应都需要由开发人员定义。而浏览器时间则不然，通常开发人员只需要定义响应事件即可，浏览器会帮你触发这些事件。

当需要在angular中定义浏览器事件的相应方法时，通常可以使用angular.element方法。这个方法也比较智能，当它判断jQuery可用时，会使用jQuery来初始化元素。否则会使用angular中自带的jqLite进行元素的初始化。

