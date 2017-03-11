---
title: '[AngularJS面面观] 7. scope继承与digest循环 - scope树结构的遍历'
date: 2016-06-12 15:53:00
categories: [源码分析, AngularJS]
tags: [源码分析, AngularJS]
---

在上一篇文章中，介绍了scope继承本质上也是基于JavaScript原型继承。同时也分析和讨论了scope生命周期中最重要的两个方法`$new`以及`$destroy`的源代码实现。

而在这一篇文章中，会接着讨论digest循环是如何利用scope的树形继承结构来进行遍历的。这也解答了在[这篇文章](http://blog.csdn.net/dm_vincent/article/details/51607018)末尾遗留下来的第一个问题。

<!-- More -->

首先，还是回顾一下scope核心方法`$digest`的实现，这里会略去一些和本节内容关系不大的实现，聚焦于scope树形结构遍历的部分：
```js
$digest: function() {
  var next, current, target = this;
  // 此处省略了其它本地变量的声明以及对applyAsync的处理
  do { 
    dirty = false;
    current = target;

    // 此处省略了对于evalAsync的处理

    traverseScopesLoop:
    do {
      // 遍历当前current对象上的watchers

      // 有点疯狂的深度优先遍历
      if (!(next = ((current.$$watchersCount && current.$$childHead) ||
          (current !== target && current.$$nextSibling)))) {
        // 深度优先遍历中回朔到父节点的过程
        while (current !== target && !(next = current.$$nextSibling)) {
          current = current.$parent;
        }
      }
    } while ((current = next));

    if ((dirty || asyncQueue.length) && !(ttl--)) {
      // 达到最大digest数量，抛出异常
    }

  } while (dirty || asyncQueue.length);

  // 此处省略了postDigestQueue的处理
}
```
上述代码中，遍历由深度优先遍历完成。这个遍历算法的代码乍一看有一些费解，可能Angular的开发者也意识到这个问题了，所以在源代码中加上了下面的注释：

> // Insanity Warning: scope depth-first traversal 
> // yes, this code is a bit crazy, but it works and we have tests to prove it! 
> // this piece should be kept in sync with the traversal in $broadcast

这段注释说这个遍历算法是有一些"疯狂"的！而且它与事件机制相关的`$broadcast`方法中用到的遍历算法是一致的。那么我们就来尝试分析一下这个算法的过程，看看是不是那么"疯狂"。

我们假设现在的scope树形结构是这样的(请无视我的绘图技术-_-)：
![假设的scope树形结构图](http://img.blog.csdn.net/20160608211222751)
同时假设每个scope上都定义有watchers。

那么在开启一轮digest循环后，定义在A(也就是rootScope)上的watchers会被遍历。

遍历结束后，就进入"疯狂"的深度优先遍历了。注意if判断本身就是
一个赋值语句：
```
next = ((current.$$watchersCount && current.$$childHead)||
(current !== target && current.$$nextSibling))
```
第一部分`(current.$$watchersCount && current.$$childHead)`的含义：
如果当前scope和及其所有孩子scope(也就是当前scope的子树)的watchers总数量大于0就取值为`current.$$childHead`，即当前scope的第一个孩子节点。注意`$$watchersCount`统计的是子树所有watchers的数量，这一点在上一篇文章介绍`incrementWatchersCount`这个方法时有提到。

第二部分`(current !== target && current.$$nextSibling)`的含义：
如果当前scope不是digest循环的起始scope(一般而言起始scope就是`rootScope`)就取值为`current.$$nextSibling`，即当前scope后面的第一个兄弟节点。

因为这两个部分通过`||`操作符进行连接，因此当第一部分满足，即当前scope存在子节点时。就会立即返回，将next指向该子节点。只有当第一部分没法满足时，才会执行第二部分。即查看当前scope是否存在后续的兄弟节点。这样就非常巧妙地实现了深度优先遍历算法中下一个执行节点的选择工作！

当next非空时，不会进入到if的内部，所以不会运行其内部的while语句。只有当next为空时(当前scope既不存在子scope，也不存在下一个兄弟scope)，才会进入if内部运行while语句，while的运行条件为：`current !== target && !(next = current.$$nextSibling)`。即当前节点不是digest起始节点并且当前节点不存在下一个兄弟节点时就会进入while循环体。而循环体内唯一的任务就是将当前节点设置为该节点的父节点。这里实现的就是深度优先遍历算法中的另一项重要任务：当没有更深的节点遍历时的回朔操作。

就这样，通过寥寥数行代码就实现了一个"疯狂"的深度优先算法。在我看来，这些代码不仅"疯狂"而且十分优美。

了解了算法执行过程后，我们以上图中的scope继承树结构为例，来看看具体的遍历顺序是如何的。

毫无疑问，首先遍历A。然后在是它的第一个孩子B。紧接着是B的第一个孩子D。D遍历结束后，发现它虽然没有孩子节点，但是有一个兄弟节点E，于是开始遍历E。随后是E的孩子F。F完毕后发现它既没有孩子节点，也没有兄弟节点。于是开始执行回朔流程，回朔到它的父亲节点E。发现E没有了下一个兄弟节点了，于是继续回朔到B。B还有一个兄弟节点C，于是停止回朔开始遍历C。C遍历结束后，由于它既没有孩子节点，也没有兄弟节点。所以回朔到它的父亲节点A。此时的next已经被设置为null/undefined	，于是外层while循环条件为false，循环终止，digest完成。

如果对上述的过程还有一些困惑，不妨再好好读读上面的代码并且注意每次执行循环判断条件后，next的指向。也许上述代码不是那么容易理解，因为它将很多的赋值语句都放在了if以及while的判断条件部分。

