---
title: '[AngularJS面面观] 12. scope中的watch机制---第三种策略'
date: 2016-07-17 23:01:00
categories: [源码分析, AngularJS]
tags: [源码分析, AngularJS, 脏检查]
---

如果你刚刚入门angular，你或许还在惊叹于angular的双向绑定是多么的方便，你也许在庆幸未来的前端代码中再也不会出现那么多繁琐的DOM操作了。

但是，一旦你的应用程序随着业务的复杂而复杂，你就会发现你手头的那些angular的知识似乎开始不够用了。为什么绑定的数据没有生效？为什么应用的速度越来越慢？为什么会出现莫名其妙的infinite digest异常？所以你开始尝试进阶，尝试弄清楚在数据绑定这个现象后面到底发生了什么。

相信能顺着前面数十篇文章看到这里的同学们，一定对angular是真爱吧。我们分析了scope中的digest循环的具体逻辑和很多细节，也见识到了angular中建立在继承树型结构之上的事件机制。本文继续聊聊关于scope的最后一个话题，watch策略。

<!-- More -->

## 基于引用以及值的watch策略

由scope中的`$watch`方法定义得到的watcher是digest循环中的工作单元。所以，神奇的双向绑定这一技术的根本之根本也就是watcher。前面已经介绍了watch机制的两种策略：

1. 比对**引用**
2. 比对**值**

而`$watch`方法提供了第三个参数用来让你选择是使用比对引用或是比对值的方式：

```js
/*
* 第三个参数objectEquality的含义：使用对象值比对，而不是引用比对
*/
$watch: function(watchExp, listener, objectEquality, prettyPrintExpression)
```

由于在angular中被双向绑定多半都是number，string这类基本类型(Primitive)，因此引用比对就是默认的选项。如果你需要比较对象(Object)，比如数组，字典对象等，那么就需要将上述$watch方法的第三个参数置为true。

定义了watcher后，那么在digest循环中就需要执行真正的比对操作了，判断一个watcher是否"脏"了的核心判断条件如下：

```js
// 用于判定一个watcher是否dirty的条件
if ((value = get(current)) !== (last = watch.last) &&
      // 上述的objectEquality即为这里的watch.eq
	  !(watch.eq
	    ? equals(value, last)
	    : (typeof value === 'number' && typeof last === 'number'
	       && isNaN(value) && isNaN(last)))) {
	dirty = true;
  // 调用watcher上的listener
  // ......
 }
```

这里利用了&&操作符的"短路"特性，首先执行`(value = get(current)) !== (last = watch.last)`，如果执行的结果为false，也就是在当前值和前值引用相等时，则判定该watcher不是"脏"的，因此也就不必要执行后续的逻辑了。由于watch的对象以基本值(Primitive)类型居多，因此这样做能够减少不必要的判断操作。当引用相同时，直接pass。只有当引用都不同时，才有可能需要"特别照顾"。if中的第二个子判断又出现了分支情况：

当需要进行值比对时，会调用angular定义的一个全局函数equals进行逐字段的递归比较过程，它能够确保被比较的两个对象真的是完全一致的，无论这个对象中嵌套了多少层对象。而第二个看似很复杂的只是为了应对JavaScript中一个很神奇的定义：`NaN === NaN // return false`。`NaN`不等于它自己，而且`NaN`的类型是number。这算是JavaScript的一个槽点吧，不过也没什么难的，遇到了仔细处理就好了。

这里还有一段示例程序，可以直接复制到本地的server上运行看看效果：

```js
<html ng-app="testApp">
<head>
  <meta charset="UTF-8">
  <title>Watch Mechanism</title>
  <script src="//cdn.bootcss.com/angular.js/1.5.7/angular.min.js"></script>
</head>
<body ng-controller="mainCtrl">
</body>
<script>
  var mod = angular.module('testApp', []);
  mod.controller('mainCtrl', function($scope, $interval) {
    $scope.obj = { 'id': 1 };

    $scope.$watch(function(scope) { return scope.obj; },
      function(newValue, oldValue, scope) {
        console.log(newValue, oldValue);
      });
      // }, true);

    $interval(function() {
      console.log('digest triggered');
      $scope.obj.id += 1;
    }, 1000);
  });
</script>
</html>
```

当在`$watch`中不传第三个参数的时候，可以看到`newValue`和`oldValue`只在控制台中被打印了一次。当传入第三个参数true的时候，可以看到每个一秒都会打印一次。

当初看到这里的时候，我有一个疑问。既然判断条件首先会判断新值和旧值的引用是否相同，再决定是否进行深度的值比对。那么上面的实例中每次改变的只是对象中的一个字段，对象本身并没有改变啊，也就意味着引用也没有被改变？

确实这个疑问大概困扰了一会。直到我继续阅读当数据"脏"了的处理逻辑时，才恍然大悟：

```js
watch.last = watch.eq ? copy(value, null) : value;
```

这里调用了定义在Angular.js中的另一个工具方法copy。现在还不着急介绍这些工具方法，简而言之这个方法会创建出一个和源对象一模一样的新对象并赋予旧值。注意这个"新"字，这也就意味着新值和旧值在每次执行值比对的时候，都是两个素昧平生的对象。因此就能够顺利地通过第一层引用判断。

以上就是我们已经见过好多次的两种watch策略。那么这第三种策略到底是何方"黑科技"，有什么存在意义和价值呢？这便是我们下面着重要分析和讨论的地方。

## 为什么需要第三种策略

一言以蔽之，因为需求和性能。

目前的两种策略是非黑即白，非此即彼的策略。

基于引用比对的策略太过于简单，对付一些基本类型还好，万一被watch的是数组，字典对象这类就完全没有用武之地了。而基于值的比对策略则又太过于凶残，对象上的任何嵌套元素都不放过，而且伴随而来的是频繁的对象深度复制操作，需要消耗CPU资源以及占用大量的内存。以至于当scope中watchers的数量比较多时会显著地影响到整体性能。

由于在一个angular应用中，被Digest Cycle关注到的并非只是一些基本类型的值(尽管这类值占了很大一部分)，有时候应用需要根据业务逻辑来关注某些数组对象或者是字典对象的变化。那么如何来找一种折中的解决方案就是这第三种策略的诞生原因。

那么这第三种策略需要支持到何种程度呢，从API文档中我们可以找到答案：

```
Shallow watches the properties of an object and fires whenever any of the properties change (for arrays, this implies watching the array items; for object maps, this implies watching the properties). If a change is detected, the listener callback is fired.

// 翻译如下
针对对象属性的浅层监视(Shallow Watch)，当属性发生变化时触发(对于数组，指的是监视数组的元素；对于字典对象，指的是监视其属性)对应listener的回调操作。
```

这里的重点是**浅层监视(Shallow Watch)**。也就是它的方案是只监视对象中的第一层元素/属性，如果这些元素/属性还有嵌套属性就不在考虑范围之内。这反映的也是一种折中的思想，如果监视的层次太深了，那么和基于值的比对策略就没多少区别了。

## 源码分析

下面开始看看angular中是如何实现这种策略的，首先我们看看源码：

```js
$watchCollection: function(obj, listener) {
  $watchCollectionInterceptor.$stateful = true;

  var self = this;
  // the current value, updated on each dirty-check run
  var newValue;
  // a shallow copy of the newValue from the last dirty-check run,
  // updated to match newValue during dirty-check run
  var oldValue;
  // a shallow copy of the newValue from when the last change happened
  var veryOldValue;
  // only track veryOldValue if the listener is asking for it
  var trackVeryOldValue = (listener.length > 1);
  var changeDetected = 0;
  var changeDetector = $parse(obj, $watchCollectionInterceptor);
  var internalArray = [];
  var internalObject = {};
  var initRun = true;
  var oldLength = 0;

  function $watchCollectionInterceptor(_value) {
    newValue = _value;
    var newLength, key, bothNaN, newItem, oldItem;

    // If the new value is undefined, then return undefined as the watch may be a one-time watch
    if (isUndefined(newValue)) return;

    if (!isObject(newValue)) { // if primitive
      if (oldValue !== newValue) {
        oldValue = newValue;
        changeDetected++;
      }
    } else if (isArrayLike(newValue)) {
      if (oldValue !== internalArray) {
        // we are transitioning from something which was not an array into array.
        oldValue = internalArray;
        oldLength = oldValue.length = 0;
        changeDetected++;
      }

      newLength = newValue.length;

      if (oldLength !== newLength) {
        // if lengths do not match we need to trigger change notification
        changeDetected++;
        oldValue.length = oldLength = newLength;
      }
      // copy the items to oldValue and look for changes.
      for (var i = 0; i < newLength; i++) {
        oldItem = oldValue[i];
        newItem = newValue[i];

        bothNaN = (oldItem !== oldItem) && (newItem !== newItem);
        if (!bothNaN && (oldItem !== newItem)) {
          changeDetected++;
          oldValue[i] = newItem;
        }
      }
    } else {
      if (oldValue !== internalObject) {
        // we are transitioning from something which was not an object into object.
        oldValue = internalObject = {};
        oldLength = 0;
        changeDetected++;
      }
      // copy the items to oldValue and look for changes.
      newLength = 0;
      for (key in newValue) {
        if (hasOwnProperty.call(newValue, key)) {
          newLength++;
          newItem = newValue[key];
          oldItem = oldValue[key];

          if (key in oldValue) {
            bothNaN = (oldItem !== oldItem) && (newItem !== newItem);
            if (!bothNaN && (oldItem !== newItem)) {
              changeDetected++;
              oldValue[key] = newItem;
            }
          } else {
            oldLength++;
            oldValue[key] = newItem;
            changeDetected++;
          }
        }
      }
      if (oldLength > newLength) {
        // we used to have more keys, need to find them and destroy them.
        changeDetected++;
        for (key in oldValue) {
          if (!hasOwnProperty.call(newValue, key)) {
            oldLength--;
            delete oldValue[key];
          }
        }
      }
    }
    return changeDetected;
  }

  function $watchCollectionAction() {
    if (initRun) {
      initRun = false;
      listener(newValue, newValue, self);
    } else {
      listener(newValue, veryOldValue, self);
    }

    // make a copy for the next time a collection is changed
    if (trackVeryOldValue) {
      if (!isObject(newValue)) {
        //primitive
        veryOldValue = newValue;
      } else if (isArrayLike(newValue)) {
        veryOldValue = new Array(newValue.length);
        for (var i = 0; i < newValue.length; i++) {
          veryOldValue[i] = newValue[i];
        }
      } else { // if object
        veryOldValue = {};
        for (var key in newValue) {
          if (hasOwnProperty.call(newValue, key)) {
            veryOldValue[key] = newValue[key];
          }
        }
      }
    }
  }

  return this.$watch(changeDetector, $watchCollectionAction);
}
```

前前后后一共130来行代码，将其中的主干逻辑提取一下后：

### 逻辑主干

```js
$watchCollection: function(obj, listener) {

  // 各类变量的声明 - 供watch函数以及listener函数的代理共享
  var changeDetector = $parse(obj, $watchCollectionInterceptor);

  // watch函数的代理
  function $watchCollectionInterceptor(_value) {
    
  }

  // listener函数的代理
  function $watchCollectionAction() {
    
  }

  return this.$watch(changeDetector, $watchCollectionAction);
}
```

抛开实现的细节，整体逻辑还是比较清晰的。首先注意看`$watchCollection`的返回：`return this.$watch(changeDetector, $watchCollectionAction)`。这说明其实在`$watchCollection`的内部还是依赖最基础的`$watch`。只不过经过了内部的各种封装，watch函数和listener函数都发生了某种变化，那么搞清楚这种变化就是我们下面的目标。

### 变量声明

第一步，看看方法的头部都声明了什么变量：

```js
$watchCollectionInterceptor.$stateful = true;

var self = this;
// 当前被关注的值，在每轮DC中会被更新
var newValue;
// 上一轮DC中newValue的浅拷贝(Shallow Copy)，在每轮DC中会被更新与newValue一致
var oldValue;
// 记录newValue的前值，是它的一份浅拷贝(Shallow Copy)
var veryOldValue;
// 是否记录veryOldValue的标志位
var trackVeryOldValue = (listener.length > 1);
// 判断watch是否触发listener的递增计数器
var changeDetected = 0;
// watch的代理
var changeDetector = $parse(obj, $watchCollectionInterceptor);
// 比对对象为数组时，oldValue的初值
var internalArray = [];
// 比对对象为字典对象时，oldValue的初值
var internalObject = {};
// listener是否首次触发的标志位
var initRun = true;
// 旧值数组对象长度的保存以及减少遍历字典对象次数的优化字段
var oldLength = 0;
```

上面列出了所有被定义在方法头部的变量，很多变量的用途和意义在没有看具体的实现之前是很难明白的，这里我事先给出了一个简要的解释，因此这里看不懂没有关系，待后面碰到了后再具体解释。

关于第一行中`$stateful`这个属性值，它其实是angular中的一个标注属性，用于告知该属性的宿主对象是带有状态的，它表示该对象会在一轮Digest Cycle中会被执行多次。反映到`$watchCollectionInterceptor`上，就是在说`watch`函数会在一轮DC中被调用多次，这符合我们已经拥有的认知。

### watch代理的实现

第二步，看看watch的代理是如何实现的：

```js
function $watchCollectionInterceptor(_value) {
  newValue = _value;
  var newLength, key, bothNaN, newItem, oldItem;

  // 针对一次性watch的处理，一次性watch和一次性绑定相关，而一次性绑定就是形如<p>Hello {{::name}}!</p>的绑定方式，在绑定后就不会参与到digest cycle中
  if (isUndefined(newValue)) return;

  if (!isObject(newValue)) {
    // 如果处理的newValue是基本类型(Primitives)，仍然采用基于引用的watch策略
    if (oldValue !== newValue) {
      oldValue = newValue;
      changeDetected++;
    }
  } else if (isArrayLike(newValue)) {
    if (oldValue !== internalArray) {
      // 将oldValue设置成一个数组对象
      oldValue = internalArray;
      oldLength = oldValue.length = 0;
      changeDetected++;
    }

    newLength = newValue.length;

    if (oldLength !== newLength) {
      // 首先比较长度，如果长度不一样则两个数组必定不同
      changeDetected++;
      oldValue.length = oldLength = newLength;
    }
    // copy the items to oldValue and look for changes.
    for (var i = 0; i < newLength; i++) {
      oldItem = oldValue[i];
      newItem = newValue[i];

      bothNaN = (oldItem !== oldItem) && (newItem !== newItem);
      if (!bothNaN && (oldItem !== newItem)) {
        changeDetected++;
        oldValue[i] = newItem;
      }
    }
  } else {
    if (oldValue !== internalObject) {
      // 将oldValue设置成一个字典对象
      oldValue = internalObject = {};
      oldLength = 0;
      changeDetected++;
    }
    // copy the items to oldValue and look for changes.
    newLength = 0;
    for (key in newValue) {
      if (hasOwnProperty.call(newValue, key)) {
        newLength++;
        newItem = newValue[key];
        oldItem = oldValue[key];

        if (key in oldValue) {
          bothNaN = (oldItem !== oldItem) && (newItem !== newItem);
          if (!bothNaN && (oldItem !== newItem)) {
            changeDetected++;
            oldValue[key] = newItem;
          }
        } else {
          oldLength++;
          oldValue[key] = newItem;
          changeDetected++;
        }
      }
    }
    if (oldLength > newLength) {
      // 旧值上的字段个数多余当前值上的字段个数，需要删除多余的
      changeDetected++;
      for (key in oldValue) {
        if (!hasOwnProperty.call(newValue, key)) {
          oldLength--;
          delete oldValue[key];
        }
      }
    }
  }
  return changeDetected;
}
```

整个watch函数代理的逻辑还是比较清晰的，分为以下几个逻辑块：

1. 处理一次性绑定的情况。形如{% raw %}<p>Hello {{::name}}!</p>{% endraw %}的绑定方式，在绑定后就不会参与到digest cycle中，从而拥有不错的性能。对于一些已经确认后就不会再改变的展示性的文字非常合适。
2. 处理基本类型。虽说`$watchCollection`主要用来处理数组和字典对象，但是来了基本类型你也不能赶人家走不是。这时直接使用基于引用的比较策略。
3. 处理"类数组类型(Array-like)"。所谓"类数组类型"，典型的比如`arguments`隐式参数列表对象以及`NodeList`这种，它们并不是真是的数组对象，不能调用诸如`slice`这样的数组方法。这类对象的共通特性是拥有length属性以及能够使用[]操作符进行随机访问。具体而言，可以参考[这里](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Functions/arguments)。
4. 处理字典对象。就是我们常见的以`{}`形式出现的对象。
5. 返回值是一个名为`changeDetected`的变量。而`changeDetected`这个变量是在整个watcher的生命周期随着值的变化而发生自增操作的，也就是说某次DC中上述watch代理返回的`changeDetected`和上一次执行返回的`changeDetected`值不相同的话，就认为被监控的值发生了变化，需要调用对应listener代理函数。这一点非常巧妙地维护了watch方法的契约。返回什么不重要，重要的是当被监控的值发生变化时，返回的值一定和上一次返回的值不一样。那么为什么需要这样做呢？在最基本的$watch方法中定义的watch函数一般返回的就是被监控的数据源本身，如果数据是基本类型还好说，是复杂的数组或者字典对象就比较折腾了，因为需要逐个地进行比对，不如比较一个基本数值来的快。这也是`$watchCollection`需要规避的，因此它采用了`changeDetected`这一变量来维持方法的契约。

重点就是如何处理"类数组类型(Array-like)"以及字典对象。

#### 处理类数组类型(Array-like)

下面是处理类数组类型的逻辑：

```js
else if (isArrayLike(newValue)) {
  if (oldValue !== internalArray) {
    // 将oldValue设置成一个数组对象
    oldValue = internalArray;
    oldLength = oldValue.length = 0;
    changeDetected++;
  }

  newLength = newValue.length;

  if (oldLength !== newLength) {
    // 首先比较长度，如果长度不一样则两个数组必定不同
    changeDetected++;
    oldValue.length = oldLength = newLength;
  }
  // 进行比对并将新的元素拷贝到oldValue上
  for (var i = 0; i < newLength; i++) {
    oldItem = oldValue[i];
    newItem = newValue[i];

    bothNaN = (oldItem !== oldItem) && (newItem !== newItem);
    if (!bothNaN && (oldItem !== newItem)) {
      changeDetected++;
      oldValue[i] = newItem;
    }
  }
} 
```

首先会调用`isArrayLike`来判断`newValue`是不是一个"类数组"。如果是第一次运行watch，那么此时`oldValue`还是`undefined`，此时将它设置为一个空的数组，并将`changeDetected`自增来保证第一运行总是会触发对应的listener函数。

然后开始比较当前数组的长度和上一次数组的长度，如果长度不一致，则自增`changeDetected`表示发生了变化，这一点可以应对在向数组中插入元素或者移除元素时产生的变化。而对于另一些数组操作，例如排序，改变元素的值等等，则可以通过一次遍历来发现是否发生了变化，在遍历的过程中顺便将`newValue`到`oldValue`的同步也完成了，而这也是为何需要设置一个`oldValue`数组的原由。由于`NaN === NaN`会得到false这个逆天的存在，也需要做一点特殊处理。

#### 处理字典对象类型(Object)

下面是处理字典对象的逻辑：

```js
} else {
  if (oldValue !== internalObject) {
    // 将oldValue设置成一个字典对象
    oldValue = internalObject = {};
    oldLength = 0;
    changeDetected++;
  }
  // 进行比对并将新的属性拷贝到oldValue上
  newLength = 0;
  for (key in newValue) {
    if (hasOwnProperty.call(newValue, key)) {
      newLength++;
      newItem = newValue[key];
      oldItem = oldValue[key];

      if (key in oldValue) {
        bothNaN = (oldItem !== oldItem) && (newItem !== newItem);
        if (!bothNaN && (oldItem !== newItem)) {
          changeDetected++;
          oldValue[key] = newItem;
        }
      } else {
        oldLength++;
        oldValue[key] = newItem;
        changeDetected++;
      }
    }
  }
  if (oldLength > newLength) {
    // 旧值上的字段个数多余当前值上的字段个数，需要删除多余的
    changeDetected++;
    for (key in oldValue) {
      if (!hasOwnProperty.call(newValue, key)) {
        oldLength--;
        delete oldValue[key];
      }
    }
  }
}
```

如果是第一次运行watch，那么此时`oldValue`还是`undefined`，此时将它设置为一个空的字典对象，并将`changeDetected`自增来保证第一运行总是会触发对应的listener函数。

然后开始执行一个循环。这个循环的作用是将`newValue`上所有的属性全部都同步到`oldValue`上，如果出现不一致则自增`changeDetected`表示数据发生了改变。同样，对`NaN`也作出了单独处理。注意在循环中只会判断`newValue`自身的属性，从`prototype`继承而来的属性不再考虑范围之内：`if (hasOwnProperty.call(newValue, key))`。在这一轮循环结束之后，`newValue`中出现的属性在`oldValue`中都会出现，但是`oldValue`中的属性可不一定会出现在`newValue`中的。比如将字典对象中删除一个属性时，就会出现这种情况。如果出现了这种情况，那么还需要以oldValue为基准再进行一次遍历，来保证`oldValue`中多余的值被移除。显而易见，并不是每次执行`watch`函数都需要第二次遍历的，那么如何来甄别出这种情况呢，答案就是上述代码中的`if (oldLength > newLength)`。顾名思义，`oldLength`和`newLength`分别是`oldValue`和`newValue`上属性的个数，如果`oldLength` 大于`newLength`，就代表此时`oldValue`上的属性比`newValue`多，需要移除`newValue`上不存在的属性来保持两者的同步。

### listener代理的实现

以上就是`$watchCollection`中watch的机制。看似有些复杂，但是理清楚了其中的逻辑后，其实也还好。
弄清楚了watch，剩下的就是listener的代理`$watchCollectionAction`。

```js
function $watchCollectionAction() {
  if (initRun) {
    initRun = false;
    listener(newValue, newValue, self);
  } else {
    listener(newValue, veryOldValue, self);
  }

  // 如果需要追踪oldValue的值，创建一个拷贝
  if (trackVeryOldValue) {
    if (!isObject(newValue)) {
      // 基本类型
      veryOldValue = newValue;
    } else if (isArrayLike(newValue)) {
      veryOldValue = new Array(newValue.length);
      for (var i = 0; i < newValue.length; i++) {
        veryOldValue[i] = newValue[i];
      }
    } else {
      veryOldValue = {};
      for (var key in newValue) {
        if (hasOwnProperty.call(newValue, key)) {
          veryOldValue[key] = newValue[key];
        }
      }
    }
  }
}
```

angular对于listener有一个约定：第一次运行时`newValue`和`oldValue`需要保持一致。
因此就有了`initRun`这个标志位，当第一运行listener时，会将`oldValue`设置成`newValue`。

然后当`trackVeryOldValue`被设置成true，也就是说应用程序开发者在声明listener时显式地声明了`oldValue`这个参数时。就会将当前`newValue`的值拷贝一份以供listener使用。

通过这一点可以知道在需要对一个数组或者字典对象使用`$watchCollection`方法进行监控的时候，如果
listener中不需要对`oldValue`进行处理，在声明listener的时候就注意只带`newValue`这一个参数就好。否则在listener被执行时还会开销一些性能和内存资源用于`oldValue`的维护。

---

至此，watch的第三种策略`$watchCollection`就介绍完毕了，相信大家对scope中的各种watch机制也有了全新的了解。在实际应用中，如果需要对数组，字典这类对象进行监控的话，不妨考虑使用它，性能比基于值的watch策略要好很多。

