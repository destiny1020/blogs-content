---
title: '[AngularJS面面观] 13. Angular工具库 --- 异常对象创建方法minErr'
date: 2016-07-24 23:04:00
categories: [源码分析, AngularJS]
tags: [源码分析, AngularJS]
---

本系列文章会讨论Angular框架除了提供scope等核心功能外，还提供了哪些功能。

作为Angular工具库这一系列文章的开篇，首先来看看但凡程序都绕不开的一个话题 - 异常。
那么Angular在异常处理方面又提供了哪些工具呢？

## 引子 - scope中是如何抛出异常的？

首先，让我们看看在定义`$rootScope`的过程中，哪些代码和异常有关：

<!-- More -->

```js
// 定义异常对象
function $RootScopeProvider() {
  var $rootScopeMinErr = minErr('$rootScope');
  // ......
}

// 下面是抛出异常的2个场景
// 1. 当Digest Cycle正在进行，不要重复启动DC
function beginPhase(phase) {
  if ($rootScope.$$phase) {
    throw $rootScopeMinErr('inprog', '{0} already in progress', $rootScope.$$phase);
  }

  $rootScope.$$phase = phase;
}

// 2. 当DC的次数超过阈值（TTL默认值为10）时，抛出我们耳熟能详的infdig异常
if ((dirty || asyncQueue.length) && !(ttl--)) {
  clearPhase();
  throw $rootScopeMinErr('infdig',
      '{0} $digest() iterations reached. Aborting!\n' +
      'Watchers fired in the last 5 iterations: {1}',
      TTL, watchLog);
}
```

下面看看在真实情况下，抛出的异常在控制台中是什么样子的。这里需要注意的是，当引用的angular是经过压缩处理后的angular.min.js时，产生的输出和引用未经压缩处理的angular.js时是不同的。(这一点也让我在对比源码和实际输出的时候有些诧异，发现很多源码中的输出在真实的浏览器控制台环境下不见了。后来才发现经过压缩的源代码会将部分输出省略)

```
// 引用angular.min.js
angular.min.js:117Error: [$rootScope:infdig] http://errors.angularjs.org/1.5.7/$rootScope/infdig?p0=10&p1=%5B%5B%7B%22ms…urn%20scope.b%3B%20%7D%22%2C%22newVal%22%3A13%2C%22oldVal%22%3A12%7D%5D%5D
    at Error (native)
    at http://localhost:10001/angular.min.js:6:412
    at m.$digest (http://localhost:10001/angular.min.js:143:281)
    at m.$apply (http://localhost:10001/angular.min.js:145:401)
    // ......

// 引用angular.js
angular.js:13708 Error: [$rootScope:infdig] 10 $digest() iterations reached. Aborting!
Watchers fired in the last 5 iterations: [[{"msg":"fn: function (scope) { return scope.a; }","newVal":7,"oldVal":6},{"msg":"fn: function (scope) { return scope.b; }","newVal":9,"oldVal":8}],[{"msg":"fn: function (scope) { return scope.a; }","newVal":8,"oldVal":7},{"msg":"fn: function (scope) { return scope.b; }","newVal":10,"oldVal":9}],[{"msg":"fn: function (scope) { return scope.a; }","newVal":9,"oldVal":8},{"msg":"fn: function (scope) { return scope.b; }","newVal":11,"oldVal":10}],[{"msg":"fn: function (scope) { return scope.a; }","newVal":10,"oldVal":9},{"msg":"fn: function (scope) { return scope.b; }","newVal":12,"oldVal":11}],[{"msg":"fn: function (scope) { return scope.a; }","newVal":11,"oldVal":10},{"msg":"fn: function (scope) { return scope.b; }","newVal":13,"oldVal":12}]]
http://errors.angularjs.org/1.5.7/$rootScope/infdig?p0=10&p1=%5B%5B%7B%22ms…urn%20scope.b%3B%20%7D%22%2C%22newVal%22%3A13%2C%22oldVal%22%3A12%7D%5D%5D
    at angular.js:68
    at Scope.$digest (angular.js:17324)
    at Scope.$apply (angular.js:17552)
    // ......

```

可见当使用angular.js时，异常的说明(也就是上述调用`$rootScopeMinErr`时传入的第二个参数)也会被打印出来。因此，建议在学习angular的时候使用未经压缩的angular.js。里面会保留你在源代码中看到的所有输出。

了解了minErr的使用场景，下面看看它是如何定义与实现的。

## minErr方法

### 异常消息的构成

首先还是从文档来直观地感受一下：

```js
/**
 *
 * 该对象用于在Angular内部输出详尽的错误信息。可以按下面的方法进行调用：
 *
 * var exampleMinErr = minErr('example');
 * throw exampleMinErr('one', 'This {0} is {1}', foo, bar);
 * ......
 */
function minErr(module, ErrorConstructor){
  // ......
}
```

minErr方法本身是接受两个参数的。第一个参数用来定义一个模块名称，比如上述的example就可以看作是一个模块的名称。它的作用是为某类异常提供一个命名空间。第二个参数是一个可能需要的自定义Error构造函数。当默认的JavaScript Error类型无法满足需求时，就可以传入一个继承自Error类型的自定义类型作为错误类型。

而minErr方法的返回也很有意思，返回的是一个函数。该函数没有定义参数列表，但是对于参数出现的顺序有它自己的规范：
1. 第一个参数表示一个code，该code和上面的模块名称进行拼接后得到某个错误的具体字符串表示，比如上述例子中的`example.one`。
2. 第二个参数是一个消息模版，它会在使用未经压缩的angular.js时显示在抛出的异常信息中。如果使用压缩的angular.min.js则不显示。
3. 第三个参数及后续参数用于替换消息模版中的占位符：`throw exampleMinErr('one', 'This {0} is {1}', foo, bar)`，其中的{0}和{1}会被分别替换成foo和bar。

该函数的返回：
```js
function minErr(module, ErrorConstructor) {
  ErrorConstructor = ErrorConstructor || Error;
  return function() {
    // message构建过程
    return new ErrorConstructor(message);
  };
}
```

由此可见该函数重要的功能就是构造用于创建错误对象的message，它的构建过程如下：

```js
return function() {
  var SKIP_INDEXES = 2;

  var templateArgs = arguments,
    code = templateArgs[0],
    message = '[' + (module ? module + ':' : '') + code + '] ',
    template = templateArgs[1],
    paramPrefix, i;

  message += template.replace(/\{\d+\}/g, function(match) {
    var index = +match.slice(1, -1),
      shiftedIndex = index + SKIP_INDEXES;

    if (shiftedIndex < templateArgs.length) {
      return toDebugString(templateArgs[shiftedIndex]);
    }

    return match;
  });

  message += '\nhttp://errors.angularjs.org/"NG_VERSION_FULL"/' +
    (module ? module + '/' : '') + code;

  for (i = SKIP_INDEXES, paramPrefix = '?'; i < templateArgs.length; i++, paramPrefix = '&') {
    message += paramPrefix + 'p' + (i - SKIP_INDEXES) + '=' +
      encodeURIComponent(toDebugString(templateArgs[i]));
  }

  return new ErrorConstructor(message);
};
});

message += '\nhttp://errors.angularjs.org/"NG_VERSION_FULL"/' +
  (module ? module + '/' : '') + code;

for (i = SKIP_INDEXES, paramPrefix = '?'; i < templateArgs.length; i++, paramPrefix = '&') {
  message += paramPrefix + 'p' + (i - SKIP_INDEXES) + '=' +
    encodeURIComponent(toDebugString(templateArgs[i]));
}
```

代码比较长，但是逻辑主线非常清晰。完全围绕着message的构建：
1. 异常代码。由module以及code组成：`'[' + (module ? module + ':' : '') + code + '] '`
2. 通过消息模版生成异常描述。这一部分是否显示根据引用的angular.js是否压缩来决定。
3. 生成异常参考链接。点击链接后会进入到angular的官网，会给一些关于异常的解释。生成链接的过程中会拼接上调用参数。

### 简单消息模板的实现
上面构建message的第一步和第三步都很清晰，重点来看看第二步是如何实现的，相关代码如下：

```js
message += template.replace(/\{\d+\}/g, function(match) {
  var index = +match.slice(1, -1),
    shiftedIndex = index + SKIP_INDEXES;

  if (shiftedIndex < templateArgs.length) {
    return toDebugString(templateArgs[shiftedIndex]);
  }

  return match;
});
```

这里运用到了replace方法的第二个参数-接受一个function表示每个match的替换逻辑，这种用法并不是很常见。一般我们会直接使用一个字符串作为第二个参数，来表示替换字符串。相关的文档可以参考[这里](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/String/replace)

就拿这个例子而言：
`'This {0} is {1}', foo, bar`

replace第二个function参数被调用时，传入的match实际上是'{0}'和'{1}'。那么通过slice(1, -1)得到的index就分别为0和1。紧接着通过index来得到arguments参数类数组中对应的参数，调用`toDebugString`进行替换。关于`toDebugString`的实现：

```js
function toDebugString(obj) {
  if (typeof obj === 'function') {
    return obj.toString().replace(/ \{[\s\S]*$/, '');
  } else if (isUndefined(obj)) {
    return 'undefined';
  } else if (typeof obj !== 'string') {
    return serializeObject(obj);
  }
  return obj;
}
```

以上，就是message的构建过程。其中实现了一个简单的模板替换算法。当需要在应用中实现类似逻辑，又不希望为这么一点功能引用一些第三方库比如[mustache](http://mustache.github.com/)，那么不妨考虑一下上述方法。另外值得一提的是，如果你的应用中已经引用了诸如[underscore](http://underscorejs.org/)或者[lodash](https://lodash.com)这样的库，这些库也提供了模版替换的方法，以[lodash中的template方法](https://lodash.com/docs#template)为例：

```js
// Use the "interpolate" delimiter to create a compiled template.
var compiled = _.template('hello <%= user %>!');
compiled({ 'user': 'fred' });
// → 'hello fred!'

// Use the HTML "escape" delimiter to escape data property values.
var compiled = _.template('<b><%- value %></b>');
compiled({ 'value': '<script>' });
// → '<b>&lt;script&gt;</b>'
```
除此之外，lodash的`template`方法还提供了更多的替换方法，详情可以参考上面的链接。

## 结语

以上就是angular中用于封装异常的方法，通过`minErr`首先构建一个某个特定模块下的异常创建方法。然后传入具体的错误代码(code)，消息模版(template)以及具体参数来完成异常对象的创建。

如果你想在你的应用中使用`minErr`来完成异常的定义，可以通过`angular.$$minErr`来得到该函数。但是从命名前面的两个`$`符号可以知道，angular并不鼓励在angular框架之外来使用它。因为随着版本的变更，作为内部实现的`minErr`函数也可能会发生变化。如果应用程序代码直接依赖于它的话，在升级angular版本的时候或许会出现一些问题(取决于`minErr`的实现和使用方式是否发生了变化)。

所以摆在我们面前有两个选项：
1. 定义属于应用程序自己的`minErr`。经过以上的讨论，可以发现它的实现原理也十分的简单清晰，因此在中大型应用中根据需要对`minErr`进行模仿和定制，创建一个应用程序版的`minErr`，以此来将日益繁杂的异常类型组织地井井有条也是一种非常不错的选择。
2. 仍然使用`angular.$$minErr`。对于中小型的应用是不错的选择，因为这类应用异常种类不会很多，使用angular框架提供的功能足矣。而且有了对`minErr`原理性的了解，只要未来的版本中不发生什么重大的变更，升级到新的版本是没有什么困难的。

