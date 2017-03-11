---
title: '[AngularJS面面观] 8. scope继承 - 属性覆盖，隔离scope以及指定scope的parent'
date: 2016-07-03 16:47:00
categories: [源码分析, AngularJS]
tags: [源码分析, AngularJS, 原型继承]
---

上一节中我们探讨了遍历scope树形继承结构的过程。本节继续讨论一下在继承结构下产生的属性覆盖问题，以及scope的一些特殊情况：隔离scope以及为scope显式指定其父亲scope。

## 属性覆盖(Attribute Shadowing)

属性覆盖这个问题或许会对Angular新手造成一定的困扰，尽管从本质上而言，它的成因还是基于JavaScript中原型继承这一概念。

那么让我们看看属性覆盖到底是怎么一回事：

```js
var parent = new Scope();
var child = parent.$new();

parent.name = 'parent name';
child.name = 'child name';
```

<!-- More -->

上述代码创建了两个scope，之间的关系一目了然。基于原型继承的机制，child其实也创建了仅属于自己的name属性，这个name属性覆盖了其parent的name属性。这个过程就叫做属性覆盖。

那么为什么这个属性覆盖有时会对新手造成困扰呢？因为在某些情况下，需要完成的操作是覆盖父scope中的属性值，习惯了Java等面向对象语言的开发者可能会直接调用`child.name = 'child name';`来试图完成覆盖。这样一来，即便是在父scope中访问name属性时，它的值就应该是child name。让我们看看实际上会发生什么：

```js
<html ng-app="testApp">
<head>
  <meta charset="UTF-8">
  <title>Document</title>
  <script src="//cdn.bootcss.com/angular.js/1.5.7/angular.min.js"></script>
  <script>
    var mod = angular.module('testApp', []);
    mod.controller('mainCtrl', function($scope) {
      $scope.name = 'parent name';
    });

    mod.controller('subCtrl', function($scope) {
      $scope.name = 'child name';
    })
  </script>
</head>
<body ng-controller="mainCtrl">
  Parent name: {{ name }}
  <div ng-controller="subCtrl">
    Child name: {{ name }} <br/>
    From Child name: {{ $parent.name }}
  </div>
</body>
</html>
```

上述代码会在浏览器中显示如下信息：

```
Parent name: parent name
Child name: child name 
From Child name: parent name
```

可见，父scope中的scope并没有被覆盖，仍然是以前的值。

出现这样的行为，让很多新手不知如何是好。这其实是不了解JavaScript原型继承机制导致的。事实上，当我们调用`child.name = 'child name';`后，child scope上也会被创建出一个新的name属性，这个name属性和父scope上的name属性除了名字相同，是没有其它任何关系的。

那么如何解决覆盖父scope上属性这一问题呢，其实有两种方案：

第一种：在子scope上调用`$scope.$parent.name = 'child name'`。

第二种：将父scope的属性放在放入到一个对象中去，然后在子scope中进行修改：

```js
$scope.user = {name: 'parent name'};  // 父scope中的声明方式

$scope.user.name = 'child name';  // 子scope中的覆盖方式
```

在实际项目代码中，第二种是更好的实现方式。这种方式也算是scope间共享数据的一种方式，尽管不是很规范。比如，可以在rootScope中定义一个对象$root，那么这个对象对所有其它scope都是可见的，因此就能够将一些公用数据放置其中。

那么为什么说这种方式不规范呢？因为随着项目规模的增大，各种类型的公用数据都可以被放入其中，久而久之其中的数据就失去了可控性。更好的实践不是通过$rootScope去共享数据，而是通过angular提供的另外一项机制：factory。factory可以被看作是一个单粒对象。在整个angular应用中只有一个实例，因此是放置公用数据的理想场所。同时，根据公用数据的类型不同，还可以针对类型声明不同的factory。

## 隔离scope
当创建一个子scope后，该scope就能够访问父scope上的属性和方法了。在大多数场景中这个特性是实用的，毕竟我们也不希望重复定义一大堆的属性和方法。但是在某些场景下，我们并不希望创建出来的scope能够自由地访问父scope中定义的属性和方法。这个时候，隔离scope就派上用场了。隔离scope从继承结构上而言，和正常的scope没什么两样，但是它的特殊之处在于它没法访问父scope上的属性和方法了。这是怎么做到的呢：

```js
var child;

parent = parent || this;

if (isolate) {
  child = new Scope();
  child.$root = this.$root;
} else {
  // non-isolated scope
  if (!this.$$ChildScope) {
    this.$$ChildScope = createChildScopeClass(this);
  }
  child = new this.$$ChildScope();
}
child.$parent = parent;
child.$$prevSibling = parent.$$childTail;
```

以上代码摘自Scope的`$new`方法。该方法的具体执行过程在介绍[scope生命周期](http://blog.csdn.net/dm_vincent/article/details/51613496)的时候讨论过。这里就不赘述了，这里是想强调创建隔离scope的过程。对于隔离scope而言，它的`$root`属性需要被指定成用来创建该隔离scope的那个scope，也就是`scope.$new(true)`中的被调用的scope。`$root`属性在哪里被用到了我暂时还并没有找到答案，至少在`rootScope`的源代码中并没有发现它的用武之地。我想也许会在其它的模块中会被用到吧，待发现之后再来更新这一部分的内容。

除了没有用原型继承来创建子scope外，隔离scope和普通的scope并没有其它的不同之处。比如parent的指向，继承数的结构设置等等。

隔离scope在自定义angular指令(directive)中用的非常多。这样做的目的也是让定义的指令更加灵活，复用性更强，不会受制于它的外部环境。由于隔离scope没法访问父scope的任何属性和方法，因此它就是一张白纸，而显然为了完成业务逻辑，就算是隔离scope也是需要数据的。这些数据的注入过程是通过directive提供的机制完成的，具体而言是通过@，=以及&三种操作符完成的。详情会在分析指令机制的时候进行分析。

## 指定scope的parent

最后，简单介绍一下如何制定scope的parent以及它有什么具体作用。`$new`方法的签名是这样的：`$new: function(isolate, parent)`。这就是说，在创建scope的时候，还能够指定待创建scope的parent。而结合前面的那段代码片段，我们可以发现parent更多的关注在scope树形继承的结构上，也就是scope之间如何组织的。而scope是否是隔离的(第一个参数isolate)，关注点在scope是否采用原型继承进行创建。这两者是两个概念，在没有阅读这段代码之前，确实非常容易将这两个概念混为一谈。

那么在什么时候需要指定scope的parent呢？目前看来，并没有什么作用。但是在后面介绍指令的transclusion(国内一些文献翻译成"透传")时，就能够发现其中的奥妙啦。
