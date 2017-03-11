---
title: '[AngularJS面面观] 3. scope中的Dirty Checking(脏数据检查) --- Digest Cycle中的优化和细节'
date: 2016-05-15 15:04:00
categories: [源码分析, AngularJS]
tags: [源码分析, AngularJS, 脏检查, 优化]
---

## Digest Cycle中的优化

在[上一篇文章](http://blog.csdn.net/dm_vincent/article/details/51407292)中，介绍了Digest Cycle的实现方法\$digest的大概逻辑。但是离真正的实现还有相当大的差距，具体的实现比较长，而且其中有很多细节在本篇文章还不会介绍，所以就不贴在这里了，有兴趣的可以去看[源码](https://github.com/angular/angular.js/blob/v1.5.5/src/ng/rootScope.js#L746)。

现在`$digest`方法的实现逻辑是这样的：
```js
var ttl = 10;
do {
  var dirty = false;
  var length = $$watchers.length;
  var watcher;
  for(var idx = 0; idx < length; idx++) {
    watcher = $$watchers[idx];
    newVal = watcher.watchFn(scope);
    oldVal = watcher.last;
    if (!newVal.equals(oldVal)) {
      watcher.last = newVal;
      watcher.listener(newVal, oldVal, scope);
      dirty = true;
    }
  }
  ttl -= 1;
  if (dirty && ttl === 0) {
    throw '10 $digest() iterations reached. Aborting!';
  }
} while (dirty);
```

<!-- More -->

### 减少watchExp执行次数的优化措施
从上面的逻辑中可以发现，DC至少会执行一遍。如果有任何listener被执行的话，那么DC至少会执行两遍。因此，当一个scope中绑定的watchers数量为1000时，最终会执行2000次与之关联的watchExp。这可是一笔不小的开销。所以我们需要尽可能地减少调用watchExp的次数。

那么如何减少？在上一篇文章中举了一个例子：
假设我们有两个watcher A和B，分别针对属性a和属性b。针对a的watcher A首先执行，如果在B的listener中改变了属性a，那么由于A已经执行过了，就不会判断出a又变“脏”了的事实。可以发现，需要多次执行DC的症结在于在后面执行的watcher中的listener可能会改变已经执行了的watcher所监控的属性。而很明显的是，即使前面执行的listener改变了某个属性，只要该属性对应的watcher在后面执行，就不会出现这个问题。

所以我们可以通过记录最后一个把数据弄“脏”的watcher，来达到减少执行次数的目的。当第二次执行这个watcher时，判断它是否是“干净的”，如果它同时也是上一轮循环中把数据弄“脏”的那个watcher。那么可以认为此轮DC到这里就结束了。下面举个例子来说明它的正确性和有效性：

假设我们有10个watcher，记为[w1, w2, ......, w9, w10]：
现在执行第一轮DC，发现w1的当前值和前值不一样。所以将它w1标记为最后一个把数据弄“脏”的watcher，记为lastDirty。在执行后续的w2到w10时，没有发现脏数据。因此不会执行它们对应的listener。
因为第一轮DC中发现了脏数据，所以按照逻辑需要执行第二次DC。此时发现w1的值没有变化，是“干净的”，同时它也是上一轮DC中被记为lastDirty的那个watcher。因此，此时就可以认为DC没有继续执行下去的不要了，终止DC。此时一共执行了11次(10 + 1)watchExp。相比优化前的20次(10 + 10)，减少了9次。

要知道这只是有10个watcher的情况，实际应用中watcher的数量只会远远超过这个值。因此平均而言该优化也能够减少`watcher总数/2`次的watchExp执行。而考虑到DC执行的相当频繁，这么一个小的优化就能够节省不少的运算资源和时间。

领会到了该优化的思想，改进代码就不是难事了。伪代码如下所示：
```
var ttl = 10;
var lastDirty;
do {
  var dirty = false;
  var length = $$watchers.length;
  var watcher;
  for(var idx = 0; idx < length; idx++) {
    watcher = $$watchers[idx];
    newVal = watcher.watchFn(scope);
    oldVal = watcher.last;
    if (!newVal.equals(oldVal)) {
      watcher.last = newVal;
      watcher.listener(newVal, oldVal, scope);
      dirty = true;
      lastDirty = watcher;
    } else if(lastDirty === watcher) {
      break;
    }
  }
  ttl -= 1;
  if (dirty && ttl === 0) {
    throw '10 $digest() iterations reached. Aborting!';
  }
} while (dirty);
```

但是，这就完事了吗？就这段逻辑，我们来和angular的实现比较一下：
```js
$watch: function(watchExp, listener, objectEquality, prettyPrintExpression) {
 ......
  lastDirtyWatch = null;
  ......
  return function deregisterWatch() {
    if (arrayRemove(array, watcher) >= 0) {
      incrementWatchersCount(scope, -1);
    }
    lastDirtyWatch = null;
  };
}
```

以上是一`$watch`方法的实现。里面也会对lastDirtyWatch这个变量进行操作。在方法中首先会将它置为空，在返回的注销函数中也会将它置为空。也就是说，当`$$watchers`数量上发生变化的时候就需要将它清空。清空意味着暂时禁用这一项旨在减少watchExp调用次数的优化。为什么需要这样做呢？

我们不妨考虑一下当在执行DC的时候，watcher数量发生变化会导致什么结果。比如当我们在某个listener中添加了一个watcher时，这个watcher的加入可能会导致某个watcher不能被执行，原因是这样的：

angular实现DC的循环体：
```js
do { // "traverse the scopes" loop
  if ((watchers = current.$$watchers)) {
     // process our watches
     length = watchers.length;
     while (length--) {
       try {
         watch = watchers[length];
         ......
```
可以发现在执行while循环的时候，循环的次数就已经明确了。即使某个listener中又添加了一个watcher，循环的次数是不会改变的。所以总会有一个watcher在这轮DC中不会执行到。而这个watcher就是发生添加watcher行为的watcher的下一个。

原本的watchers数组是这样的：[w1, w2, w3, w4, w5]。
现在在w2的listener中添加了一个新的watcher：w6。由于添加watcher采用的是unshift方法，此时的watchers数组是这样的：[w1, w2, w3, w4, w5, w6]。而循环次数是一定的，5次。所以在执行完了w2之后，还需要循环3次，按照实现逻辑，执行的是w3，w4和w5。可以发现w6就这样被活生生的忽略掉了。而无论后面执行的w3，w4和w5是否是“脏的”，被记录的lastDirty最多只可能是w5。因此在执行第二轮DC时，在运行到w5时发现lastDirty此时已经稳定了，所以判定DC可以结束运行。因此w6永远都没有被执行的机会。所以当watchers数量发生变化的时候，有必要暂时禁用此项优化，保证所有的watcher都有机会得到执行。

### 处理NaN带来的infinite digest loop问题
了解JavaScript语法规则的同学们对下面这个判断应该都有印象：
```js
NaN === NaN // false
```
当时看到false这个结果时，差点没吐出一口老血。为什么自己和自己比较还可以不想等。
虽然用到这个结论的场合很有限，但是当切换到angular这个场景中时，没有恰当处理的话可能会抛出你想象不到的错误。

毕竟，是否执行listener的依据就是比较两个值是否相等。而如果不对NaN做特殊处理那么每次都会调用listener，watcher永远没有稳定的那一天。结果就是不断的抛出`10 $digest() iterations reached. Aborting!`

因此，需要对NaN单独做出判断：
```js
typeof value === 'number' && 
  typeof last === 'number'&& 
  isNaN(value) && isNaN(last)
```
当满足上面的判断时，也认为两个值是相等的。这样就不会造成错误地调用listener了。

### 使用reversed while循环方式进行watchers的遍历
循环的方式我们一般认为倒序的会快一点，同时while循环也比for循环要快一点。这里有[相关讨论](http://stackoverflow.com/questions/1340589/are-loops-really-faster-in-reverse)，有兴趣的同学可以看看。

所以在angular的实现中，也采用的是逆序遍历结合while循环的方式。具体到我们的伪代码中：
```js
var ttl = 10;
var lastDirty;
do {
  var dirty = false;
  var length = $$watchers.length;
  var watcher;
  while(length--) {
    watcher = $$watchers[length];
    newVal = watcher.watchFn(scope);
    oldVal = watcher.last;
    if (!newVal.equals(oldVal)) {
      watcher.last = newVal;
      watcher.listener(newVal, oldVal, scope);
      dirty = true;
      lastDirty = watcher;
    } else if(lastDirty === watcher) {
      break;
    }
  }
  ttl -= 1;
  if (dirty && ttl === 0) {
    throw '10 $digest() iterations reached. Aborting!';
  }
} while (dirty);
```

当然，除了性能方面的考量外。使用逆序遍历还有一个很重要的考虑：当一个watcher将其自身移除后，不要影响到后续watcher的执行！

举个例子：
现在有三个watchers：[A, B, C]。在一轮DC时，按照A，B，C的顺序执行。当在执行B的时候，它会将它自身注销掉：

```js
var deregisterB = scope.$watch(function() {
  deregisterB();
});
```

在watchExp中执行移除操作后，数组就变成了：[A, C]。
由于执行B的时候执行循环的索引是1，当执行完毕之后索引变为2，此时判断索引和当前数组的长度相等。因此循环结束，就这样C并没有被执行到！

然而，当我们采用逆序遍历后，就不存在这个问题了：
首先执行C，然后在执行B后，循环的索引是0。此时索引0对应的元素正好是C，因此C仍然会被执行。

### 不暴露watcher的初值---initWatchVal
另外一个细节是，在目前的listener处理逻辑中，watcher的初值是会被暴露出去的。在上一篇文章中，我们讨论了angular为了保证第一次执行watchExp时总会触发listener，会将初值设置为一个function。而显然我们在触发listener时，不需要将该function作为oldValue暴露出去：

```js
watcher.listener(newVal, ((last === initWatchVal) ? newVal: oldVal), scope);
```

当前值等于初始值时，直接将当前值作为前值作为参数调用listener。

### 异常处理
目前，DC的核心代码还不够健壮，如果调用watchExp或者listener的过程中出现了异常，那么整个DC就跪了。这显然是不行的，所以需要添加try-catch来保证即使某个watcher出现了问题，也不会影响到其它的watchers。加上了异常处理及以上各种细节后的DC核心代码如下所示：

```js
var ttl = 10;
var lastDirty;
do {
  var dirty = false;
  var length = $$watchers.length;
  var watcher;
  while(length--) {
    try {
      watcher = $$watchers[length];
      newVal = watcher.watchFn(scope);
      oldVal = watcher.last;
      if (!newVal.equals(oldVal)) {
        watcher.last = newVal;
        watcher.listener(newVal, ((last === initWatchVal) ? newVal: oldVal), scope);
        dirty = true;
        lastDirty = watcher;
      } else if(lastDirty === watcher) {
        break;
      }
    } catch(e) {
      console.error(e);
    }
  }
  ttl -= 1;
  if (dirty && ttl === 0) {
    throw '10 $digest() iterations reached. Aborting!';
  }
} while (dirty);
```

在下一篇文章中，会介绍和DC中`$digest`方法息息相关的方法：`$apply`以及`$eval`。

感谢大家花费宝贵时间阅读我的文章，如果发现文中有不妥之处，请赐教！谢谢大家。


