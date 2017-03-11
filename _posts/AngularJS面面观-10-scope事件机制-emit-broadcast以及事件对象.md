---
title: '[AngularJS面面观] 10. scope事件机制 - $emit, $broadcast以及事件对象'
date: 2016-07-10 22:50:00
categories: [源码分析, AngularJS]
tags: [源码分析, AngularJS, 事件机制]
---

在上一篇文章中，介绍了事件机制背后的订阅-发布模式以及angular事件的生命周期。

本文继续介绍和事件机制相关的几个重要组成部分：

1. 事件对象的组成
2. 事件与scope继承树-`$emit`以及`$broadcast` 
3. 事件的停止传播以及阻止默认行为

<!-- More -->

## 事件对象的组成

在调用事件回调函数的时候，事件对象会被构建出来并传入到回调函数中去。那么这个事件对象包含了哪些字段呢？由于事件的触发入口只有下面将会介绍的`$emit`以及`$broadcast`，所以弄清楚在相应代码中事件对象是如何构建出来的就可以了：

```js
// $emit
$emit: function(name, args) {
  var empty = [],
      namedListeners,
      scope = this,
      stopPropagation = false,
      event = {
        name: name,
        targetScope: scope,
        stopPropagation: function() {stopPropagation = true;},
        preventDefault: function() {
          event.defaultPrevented = true;
        },
        defaultPrevented: false
      },
      listenerArgs = concat([event], arguments, 1),
      i, length;

  // ...
}

// $broadcast
$broadcast: function(name, args) {
  var target = this,
      current = target,
      next = target,
      event = {
        name: name,
        targetScope: target,
        preventDefault: function() {
          event.defaultPrevented = true;
        },
        defaultPrevented: false
      };

  if (!target.$$listenerCount[name]) return event;
  var listenerArgs = concat([event], arguments, 1),
      listeners, i, length;

  // ...
}
```

两者的构建方法略有差别，但是共通的部分也不少，列举如下：

1. name：事件的名称，起的作用相当于是key，是scope中两个事件相关字典对象的key。
2. targetScope：初始值都被设为当前scope。
3. preventDefault以及defaultPrevented：是一个函数。调用后会将本来为false的defaultPrevented置为true。

另外在`$emit`中的事件对象还有一个`stopPropagation`函数。用来将`stopPropagation`标志位设置为true。

在构建完成基本的事件对象后，还会根据该对象和实际传入到`$emit`以及`$broadcast`中的参数构建出一个数组对象，这个数组对象才是真正会被传入到回调函数中的参数：`listenerArgs`。

这个concat函数定义在Angular.js中：
```js
// 其中的slice对象就是Array类型上的slice方法
function concat(array1, array2, index) {
  return array1.concat(slice.call(array2, index));
}
```

因此`concat([event], arguments, 1)`的作用就是构建出一个形为[event, arg1, arg2, arg3, ...]的数组。
如果对JavaScript函数中的arguments参数作用不熟悉，可以参考这篇文章：[MDN官方文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Functions/arguments)。

除了在event对象构建之初就能够确定的`targetScope`，在事件在scope树形结构中流转的时候还会在event对象上创建另外一个名为`currentScope`的字段。这两个字段的命名完全是参照DOM事件的命名方式：

DOM事件：事件发生的DOM节点为`target`，事件捕获/冒泡过程中的流转经由DOM节点为`currentTarget`。
Angular事件：事件发生的scope为`targetScope`，事件向上传递/向下广播过程中流转经由的scope为`currentScope`。

这样对比一下是不是一目了然呢。

## 事件与scope继承树

为了继续讨论后面的内容，我画了一张scope树形结构图作为例子(还是请忽略我的绘图技术，随手画的。囧)。
![这里写图片描述](http://img.blog.csdn.net/20160703213054925)

该结构由5个节点组成：其中4是一个隔离scope(其中的I表示Isolated)。
并假设每个节点上都注册有一定数量的事件。

### 向上传递的$emit

创建好了事件对象，下面来看看事件是如何向上传递的，在传递的过程中有哪些值得留意的行为：

```js
$emit: function(name, args) {
  // 创建事件对象以及各种变量的声明
  // ......

  // 遍历开始
  do {
    namedListeners = scope.$$listeners[name] || empty;
    event.currentScope = scope;
    for (i = 0, length = namedListeners.length; i < length; i++) {
      // 如果存在被注销的回调函数，则整理回调函数数组以消除null元素
      if (!namedListeners[i]) {
        namedListeners.splice(i, 1);
        i--;
        length--;
        continue;
      }
      try {
        // 执行当前scope上注册的所有name对应的回调函数
        namedListeners[i].apply(null, listenerArgs);
      } catch (e) {
        $exceptionHandler(e);
      }
    }
    // 如果任何回调设置了stopPropagation，那么终止冒泡过程
    if (stopPropagation) {
      event.currentScope = null;
      return event;
    }
    // 向上遍历
    scope = scope.$parent;
  } while (scope);

  event.currentScope = null;

  return event;
}
```

值得留意的有以下几个地方：

1. 处理回调函数中空元素的逻辑。首先想想什么情况下才会出现这种情况呢？纵观和事件相关的代码，发现只有在注销的时候才会将数组元素置空。那么在遍历的过程中为什么需要处理呢？难道遍历中会发生事件的注销吗？答案是：是的，在回调函数就有可能把它自己给注销了。当只需要调用一次某个回调函数的时候，就会出现这种情况。
2. 在以此遍历每个回调函数的时候：`namedListeners[i].apply(null, listenerArgs)`，传入的参数都是listenerArgs。也就是说，如果第一个回调函数改变了event或者是其它参数，后续的回调函数就能够发现并根据参数作出合适的处理。这个特性在一些场景下会有用武之地，比如第一个回调如果计算得到了一个值，就可以将该值放入到参数中供后续的回调函数使用。
3. `preventDefault`这个flag并没有在遍历过程中被使用，这个flag可以在回调函数中使用，根据其值执行不同的业务逻辑。也可以在其它需要的地方使用，因为它也是返回的事件对象上的一个属性，这一点和`stopPropagation`不一样，后者并不是事件对象上的属性。
4. 返回event对象之前，会清空其中定义的`currentScope`属性。因为该属性随着遍历会发生变化，因此将它暴露出去没有意义，在返回之前清空。
5. 检测是否`stopPropagation`的逻辑发生在循环当前scope的所有回调之后。这样做能够保证当前scope上的所有回调都会被执行。

拿之前的scope继承结构作为例子，当在4号隔离scope上调用$emit时，遍历的顺序是4->2->1。

### 向下广播的$broadcast

`$broadcast`的整体结构也非常清晰：

```js
$broadcast: function(name, args) {
  // 创建事件对象以及各种变量的声明
  // ......

  if (!target.$$listenerCount[name]) return event;

  while ((current = next)) {
    event.currentScope = current;
    listeners = current.$$listeners[name] || [];
    for (i = 0, length = listeners.length; i < length; i++) {
      // 如果存在被注销的回调函数，则整理回调函数数组以消除null元素
      if (!listeners[i]) {
        listeners.splice(i, 1);
        i--;
        length--;
        continue;
      }

      try {
        listeners[i].apply(null, listenerArgs);
      } catch (e) {
        $exceptionHandler(e);
      }
    }

    // 和digest循环中一样的深度优先遍历(DFS)
    // 不同点：会检查$$listenerCount
    if (!(next = ((current.$$listenerCount[name] && current.$$childHead) ||
        (current !== target && current.$$nextSibling)))) {
      while (current !== target && !(next = current.$$nextSibling)) {
        current = current.$parent;
      }
    }
  }

  event.currentScope = null;
  return event;
}
```

值得留意的有以下几点：

1. `$emit`值得留意的几个点的前四个，在向下广播的过程中同样出现了。
2. 不可`stopPropagation`：和`$emit`不一样的是，在`$broadcast`的过程中，不可以终止遍历。这也许和深度优先遍历的算法特点相关，`currentScope`的流转过程并没有像`$emit`中那么清晰。所以贸然地设置`stopPropagation`并没有多少意义。所以angular干脆就在`$broadcast`中不提供这个功能了。
3. 遍历的方式为深度优先遍历(DFS)，关于这种遍历方式的讨论，在[这篇文章](http://blog.csdn.net/dm_vincent/article/details/51614721)中进行了详尽描述。但也不是任何时候在某个scope上调用$broadcast，就会跑一边以该scope为根节点，所在子树的遍历的。这个时候前面介绍的回调函数的计数器字典对象就派上用场了：`if (!target.$$listenerCount[name]) return event;`只有当该子树拥有的对应回调函数数量大于0的时候，才会遍历。这也算是性能上的一个小优化吧。否则在没有注册回调函数的情况下，每次都遍历只会浪费性能。

## 事件的停止传播以及阻止默认行为

关于这一点，其实在上面介绍`$emit`和`$broadcast`方法的时候，就已经提及了。这里再做一次总结：
1. `$emit`在遍历过程中可以让事件停止传播，但`$broadcast`的遍历不行。
2. 是否阻止了默认行为在事件机制本身中并不会被用到，但是由于它是事件对象上的一个属性，而事件对象在调用了`$emit`和`$broadcast`后都会被作为返回值返回。因此应用程序逻辑可以根据该属性做出合适的处理。

---

至此关于angular中的事件机制就介绍完毕了。
下一篇文章会介绍事件机制在angular框架中的应用。



