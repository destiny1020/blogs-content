---
title: '[AngularJS面面观] 11. scope事件机制 - 事件系统在Angular框架中的应用'
date: 2016-07-16 22:54:00
categories: [源码分析, AngularJS]
tags: [源码分析, AngularJS, 事件机制]
---

此篇文章是angular事件机制相关的最后一篇文章。
主要介绍一下事件系统在Angular框架本身中的一些应用场景，看看在什么场景下使用事件是比较合适的。

## 移除scope后的广播
有过定义指令(directive)经验的同学们应该知道，很多指令都会拥有自己的scope，无论是隔离scope也好，还是原型继承的scope也好。这些指令在浏览器中也是通过对应模板(template)所表示的DOM元素来刷存在感的。而显然，伴随业务逻辑的执行，一些指令对应的DOM元素会被创建，使用和销毁。

<!-- More -->

DOM的创建就意味着资源的分配，比如DOM元素的样式修饰，事件的绑定等等。而这些资源并不是无限的，因此当该DOM被销毁的时候，就需要有办法能够回收这些资源。典型的比如jQuery中的`bind`方法和`unbind`方法就是干的事件回调函数这一资源的分配和回收。那么在angular这一框架中，是如何处理scope的回收的呢？由于本身就自带事件系统，当然是使用它自己的更接地气啦。而且前面的文章中也提到了，angular的事件机制和scope的继承树型结构是融为一体的。那么使用它自己的事件来处理回收这一问题就顺风顺水了。

具体而言，是如何做到呢？

首先，在创建scope的时候，就会监听一个名为`$destroy`的事件。注意这个时间名称以$开头，表明这是一个angular内部的事件名称，这一点和其它angular方法如`$apply`，`$digest`类似。

```js
$new: function(isolate, parent) {
  // 构建新的scope，定义为child
  // ......

  // 为什么只有隔离scope和parent被显示指定的情况下才需要注册$destroy的回调？
  // 因为回调函数destroyChildScope中的唯一操作就是将当前scope上的$$destroyed属性置为true
  // 而这个$$destroyed属性在其它情况下都会通过原型继承机制共享给当前scope的所有孩子
  if (isolate || parent != this) child.$on('$destroy', destroyChildScope);

  return child;
}

function destroyChildScope($event) {
  $event.currentScope.$$destroyed = true;
}
```

除了在创建scope时就未雨绸缪考虑到了将来删除这个scope时需要调用的回调函数外，在应用程序代码的任何地方也能够根据需要为某个scope创建一个删除事件(`$destroy`)的回调。举个例子，当我们创建并打开一个模态对话框后，当用户关闭这个对话框后，对话框中的资源都需要被回收。如果你的模态对话框中使用了`$timeout`或者`$interval`服务的话，那么监听删除事件就是一个绝佳的用来回收这些资源的方式。

这只是事件系统的一环，一个巴掌拍不响，注册了用于监听的回调函数。还需要有个地方触发。对于scope的删除事件，触发它的任务就自然而然落在了`$destroy`方法身上：

```js
$destroy: function() {
  // 避免重复删除scope
  if (this.$$destroyed) return;
  var parent = this.$parent;

  // 广播$destroy事件后将标志位$$destroyed置为true
  this.$broadcast('$destroy');
  this.$$destroyed = true;

  if (this === $rootScope) {
    // 应用程序终止的清理工作
    $browser.$$applicationDestroyed();
  }

  // 更新各种事件的回调函数数量
  incrementWatchersCount(this, -this.$$watchersCount);
  for (var eventName in this.$$listenerCount) {
    decrementListenerCount(this, this.$$listenerCount[eventName], eventName);
  }

  // 调整scope树结构，剥离当前被删除scope的存在感 --- 一些指针操作
  if (parent && parent.$$childHead == this) parent.$$childHead = this.$$nextSibling;
  if (parent && parent.$$childTail == this) parent.$$childTail = this.$$prevSibling;
  if (this.$$prevSibling) this.$$prevSibling.$$nextSibling = this.$$nextSibling;
  if (this.$$nextSibling) this.$$nextSibling.$$prevSibling = this.$$prevSibling;

  // 将各种方法无效化，防止错误调用。清除回调函数字典对象
  this.$destroy = this.$digest = this.$apply = this.$evalAsync = this.$applyAsync = noop;
  this.$on = this.$watch = this.$watchGroup = function() { return noop; };
  this.$$listeners = {};

  // 继续调整scope树结构，cleanUpScope针对IE9作了一些特殊处理，具体请参考相关代码
  this.$$nextSibling = null;
  cleanUpScope(this);
}
```

代码中的注释简单解释了一下对应部分代码起到的作用。具体的清理过程就不继续深究了，感兴趣的同学可以仔细阅读一下相关源代码。

下面我们看看在什么地方会调用这个定义在scope对象上的`$destroy`方法。简单的一顿全局搜索，发现很多angular内置指令都会调用它，比如我们经常使用的`ngIf`指令，摘录部分代码如下

```js
if (value) {
  // 当ng-if后表达式解析为true时执行的代码
} else {
  // 当ng-if后表达式解析为false时执行的代码
  // ......
  if (childScope) {
    childScope.$destroy();
    childScope = null;
  }
  // ......
}
```

那么当ng-if后表达式解析为false时，就会删除它对应的scope。所以这也意味着如果ng-if表达式的值变化的比较频繁，那么scope的创建和销毁也会比较频繁。如果在你的应用代码中使用了非常多的ng-if而发现应用有时卡顿比较明显的话，不妨考虑一下使用ng-show这一指令，它并不会导致频繁的scope创建和销毁。

-----------------------------------未完待续------------------------------------------

随着这一系列文章的更新，我们也会了解越来越多angular内部的实现原理。如果其中用到了事件机制，在这篇文章中也会继续更新。

敬请期待。
