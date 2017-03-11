---
title: '[AngularJS面面观] 17. 依赖注入 --- 注解的定义与实现'
date: 2016-08-07 00:22:00
categories: [源码分析, AngularJS]
tags: [源码分析, AngularJS, 依赖管理, 注解]
---

本篇文章继续介绍angular用以实现依赖注入的关键元素之一 - 注解(Annotation)。

在前几篇文章中，我们已经分析和讨论了有关angular依赖注入的几个方面：

1. [angular如何处理模块的声明和获取](http://blog.csdn.net/dm_vincent/article/details/51884464)
2. [angular注入器的概念和它是如何加载模块以及执行模块定义的任务](http://blog.csdn.net/dm_vincent/article/details/52015678)
3. [angular注入器如何管理被托管的对象](http://blog.csdn.net/dm_vincent/article/details/52073838)

既然我们定义的服务和数据都已经被angular注入器托管在其内部的缓存中了，接下来应该如何使用它呢？写过angular应用的同学们应该都写过下面这类代码：

<!-- More -->

```js
var testApp = angular.module('test', []);

testApp.controller('testController', function($scope, $rootScope) {
  // ......
});
```

我们直接在`testContoller`的函数中定义了两个服务，`$scope`以及`$rootScope`，然后就能够在应用的业务逻辑中使用这两项服务了，不需要自己将它们创建出来，也不需要做什么特别的准备工作，一切都显的水到渠成。但是！程序开发并不是变魔法，看似不可思议的事情背后总是有支撑它发生的逻辑。我们都知道这个逻辑就是依赖注入，而依赖注入的关键在于注入器，所以问题就演变成了注入器是如何找到对应的被托管的对象的呢？

答案是通过注解(Annotation)。所谓注解，它的本质就是给源代码添加一些元数据。有Java开发经验的同学想必都见过`@Override`，`@Deprecated`以及`@SuppressWarnings`这类常用注解吧。它们的意义分别是表明某个方法覆盖了/实现了父类型(可以是父类，也可以是接口)上的同名方法；表明某个方法已经废弃了，不推荐再使用；抑制编译器产生警告信息。因为这类信息十分必要，但是又不好直接以传统意义上的源代码的形式体现出来，所以才设计出了注解这种数据类型。

那么切换到angular的上下文中，又是如何来实现注解的呢？这个注解需要解决什么问题呢？这就是本篇文章需要分析和讨论的主题。

我们已经知道定义的各种服务的函数实际上并不由我们自己来调用，而是交给angular框架提供的注入器进行调用，在调用的过程中首先我们需要知道注入器在哪个阶段会需要使用到注解提供的信息。其实在上一篇文章中在介绍注入器实例化托管对象的过程中可能发生循环依赖异常时就已经有一些线索了，这些线索就隐藏在异常的调用栈之中，我们来看看：

```
angular.js:13920 Error: [$injector:cdep] Circular dependency found: service1 <- service2 <- service1
http://errors.angularjs.org/1.5.8/$injector/cdep?p0=service1%20%3C-%20service2%20%3C-%20service1
    at angular.js:68
    at getService (angular.js:4656)
    at injectionArgs (angular.js:4688)
    at Object.instantiate (angular.js:4730)
    at Object.<anonymous> (angular.js:4573)
    at Object.invoke (angular.js:4718)
    at Object.enforcedReturnValue [as $get] (angular.js:4557)
    at Object.invoke (angular.js:4718)
    at angular.js:4517
    at getService (angular.js:4664)
```

可以看到在几个调用点： `Object.instantiate` -> `injectionArgs` -> `getService`。

毫无疑问，从字面意思上就能够理解这里发生了什么。首先注入器会尝试实例化一个被托管的对象，在实例化的过程中由于该对象也存在依赖关系，需要首先解析这些依赖关系，得到依赖关系之后，才能够调用`getService`完成真正的实例化操作。而解析依赖关系实际上就是我们所关注的注解的生成过程。那么在清楚了注解的应用场景后，让我们看看`injectionArgs`这个函数的逻辑是怎样的：

```js
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
    // 通过注解的key来得到的真正依赖的对象
    args.push(locals && locals.hasOwnProperty(key) ? locals[key] :
                                                     getService(key, serviceName));
  }
  return args;
}
```

这段代码完成了几件事情：

1. 通过`createInjector.$$annotate`来得到所调用函数的注解信息`$inject`。这是我们需要关注的重点。
2. 遍历注解信息`$inject`，确保每个元素都是字符串类型(即被依赖的托管对象的名字)，如果存在别的类型将直接抛出异常。
3. 通过`$inject`中的被依赖托管对象的名字来得到真正的被托管对象。最后将这些对象返回。

因此如何构建`$inject`就是问题的关键所在。构建`$inject`的过程被封装到了`createInjector.$$annotate`表示的函数中，而这个函数的实现在injector.js中，代码如下所示：

```js
// 在injector.js的最后一行定义了如下代码：createInjector.$$annotate = annotate
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

看看上述代码的整体逻辑，可以发现`$inject`的构建大概有几种方式：

1. 直接给fn提供`$inject`属性。
2. `fn`类型是函数且没有`fn`上没有`$inject`这个属性并且`strictDi`不为true时，通过一段操作来得到$inject`。
3. 当`fn`为数组类型时，截取它的前n-1个元素作为 `$inject`。

这三种方式即为angular中注解的几种声明和工作方式。下面我们逐一进行介绍：

1. 直接提供`$inject`属性

示例代码如下所示：

```js
var testApp = angular.module('test', []);

testApp.controller('testCtrl', testCtrlFunc);

testCtrlFunc.$inject = ['aConstant', 'bConstant'];
function testCtrlFunc(a, b) {
  // a 代表的就是 aConstant
  // b 代表的就是 bConstant
}
```

这种方式简单粗暴，但是由于它还需要给函数附加一个属性，导致实际中很少用到。正是因为这一点，才有了第二种基于数组的注解声明方式的诞生。它将原来的函数替换成一个数组，数组的前n-1个元素表示的就是`$inject`的信息，最后一个元素为函数本身。

因此使用这种声明方式的代码是这个样子的：

```js
var testApp = angular.module('test', []);

testApp.controller('testCtrl', ['aConstant', 'bConstant', testCtrlFunc]);

function testCtrlFunc(a, b) {
  // a 代表的就是 aConstant
  // b 代表的就是 bConstant
}
```

其实也是换汤不换药，换了个马甲就出来骗人了。这样的写法好处缩短了一点点代码量，代码也更加紧凑了。但是这还是满足不了懒人程序员们的需求，还是太麻烦了。

于是第三种方式横空出世。在初学angular的时候，我们会写这样的代码：

```js
var testApp = angular.module('test', []);

// 假设我们已经定义过了aConstant以及bConstant
testApp.controller('testController', function(aConstant, bConstant) {
  // ......
});
```

这里我们既没有为控制器的函数声明`$inject`，也木有使用基于数组的注解声明方式。但是我们还是能用上`aConstant`以及`bConstant`，这不是"黑魔法"是什么？我们来看看这个"黑魔法"背后是什么逻辑在支撑着它，重点就在`annotate`函数中的下面这一段：

```js
argDecl = extractArgs(fn);
forEach(argDecl[1].split(FN_ARG_SPLIT), function(arg) {
  arg.replace(FN_ARG, function(all, underscore, name) {
    $inject.push(name);
  });
});

// extractArgs函数的定义
function extractArgs(fn) {
  var fnText = Function.prototype.toString.call(fn).replace(STRIP_COMMENTS, ''),
      args = fnText.match(ARROW_ARG) || fnText.match(FN_ARGS);
  return args;
}

// 使用到的各种正则表达式
var ARROW_ARG = /^([^\(]+?)=>/;
var FN_ARGS = /^[^\(]*\(\s*([^\)]*)\)/m;
var FN_ARG_SPLIT = /,/;
var FN_ARG = /^\s*(_?)(\S+?)\1\s*$/;
var STRIP_COMMENTS = /((\/\/.*$)|(\/\*[\s\S]*?\*\/))/mg;
```

这个功能初看上去严重依赖于正则表达式。的确，它的总体思路是去解析函数的源代码，从其中提取出参数列表，然后通过参数列表来构建出需要的`$inject`注解信息。

来看看具体的实现过程是怎么样的：

1. 首先，需要得到函数的源代码并将参数列表解析出来。

	```js
	// 通过Function.prototype.toString.call(fn)得到函数的源代码，然后去除掉其中的注释
	var fnText = Function.prototype.toString.call(fn).replace(STRIP_COMMENTS, ''),
	// 通过ARROW_ARG或者FN_ARGS解析得到参数列表并返回
	      args = fnText.match(ARROW_ARG) || fnText.match(FN_ARGS);
	  return args;
	```
	
	那么在获取函数的源代码时，为什么调用的是`Function.prototype.toString.call(fn)`，而不是直接调用`fn.toString`呢？这里涉及到了一些JavaScript原型继承的概念。我们需要考虑一种情况，如果`fn`上重新定义了`toString`这个属性那不就没法得到源代码了嘛？所以，使用`Function.prototype.toString`能够保证调用的是函数类型的原型对象上的那个最原始的`toString`方法。
	
	得到源代码后，通过将匹配到的注释代码替换成为''来完成注释的删除工作。有了不含有注释的源代码，就可以进一步通过匹配箭头函数参数列表的正则表达式以及常规函数参数列表的正则表达式来完成参数列表的解析了。
	
	关于箭头函数，它实际上是ES6中的定义的规范之一，可以参考[MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Functions/Arrow_functions)关于箭头函数的说明。它从形式上很接近Lambda表达式，这也是近些年来很多编程语言都会添加的特性之一，也是为了迎合日渐流行的函数式编程。而常规函数不用多说，就是我们最常见的那种定义函数的方式。箭头函数和常规函数的示例如下所示：
	
	```js
	// 箭头函数
	(aConstant, bConstant) => { 
	  // ......
	}
	
	// 常规函数
	function(aConstant, bConstant) {
	  // ......
	}
	```

2. 进一步解析参数列表，得到每个参数对应被托管对象的名字。

	```js
	// argDecl就是匹配了参数列表的结果
	argDecl = extractArgs(fn);
	forEach(argDecl[1].split(FN_ARG_SPLIT), function(arg) {
	  arg.replace(FN_ARG, function(all, underscore, name) {
	    $inject.push(name);
	  });
	});
	```
	
	注意上面调用了`argDecl[1].split(FN_ARG_SPLIT)`来得到由所有参数组成的一个数组。为什么这里取的是`argDecl`的第二个元素呢？是因为`ARROW_ARG`也好，`FN_ARGS`也好，都定义了分组。第一组才是真正匹配上的参数列表的那一部分。所以需要基于`argDecl[1]`来调用`split`方法。关于正则表达式的用法，可以参考很多文档，比如[MDN的文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Regular_Expressions)，这里就不赘述了。得到了参数数组后，还需要一些处理才能够得到真正的被托管对象名字。这个处理主要是通过`FN_ARG`这一正则表达式完成的，在这个表达式中定义了两个组，第一个组匹配可能出现的下划线，第二个组匹配的才是被托管对象的名字信息。所以我们可以看到上述代码中使用了`replace`方法不那么常见，但是功能非常强大的一个重载，具体文档可以参考[MDN replace文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/String/replace)中"指定一个函数作为参数"这一部分。传入到`replace`方法中的第二参数的函数签名是这样的：`function(all, underscore, name)`，`all`代表的是匹配的整个字符串，`underscore`和`name`分别代表第一组和第二组。
	
	为什么需要处理下划线呢？这是基于angular的一条约定：如果参数被一个下划线字符包围，那么首先需要去掉包围它的下划线，剩下的部分才作为被托管对象的名字。注意，单侧的下划线不会被去掉。举个例子：
	
	```js
	var testApp = angular.module('test', []);
	
	// 假设我们已经定义过了aConstant，bConstant以及cConstant
	testApp.controller('testController', function(_aConstant_, _bConstant, cConstant_) {
	  // ......
	});
	```
	
	参数列表`['_aConstant_', '_bConstant', 'cConstant_']`会被解析成`['aConstant', '_bConstant', 'cConstant_']`。由于我们只定义了`bConstant`以及`cConstant`，而没有定义`_bConstant`和`cConstant_`。所以在尝试注入的时候是会抛出异常的。其实我并不清楚这样处理的意义在哪里，加个下划线是干嘛的呢？不过既然程序是这样写的，想必应该是有什么道理的吧。

3. 得到最终的注解信息`$inject`。

---

得到了注解信息后，再来看这段代码：

```js
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
    // 通过注解的key来得到的真正依赖的对象
    args.push(locals && locals.hasOwnProperty(key) ? locals[key] :
                                                     getService(key, serviceName));
  }
  return args;
}
```

该函数返回的`args`实际上就是将`$inject`一一转换后得到的被注入器托管对象的数组。也就是真实的参数列表。至此就完成了参数的准备工作，注入器终于可以开心地调用我们定义的函数了。

最后，我们来比较一下这三种注解方式。
首先，第一种注解方式很朴素，注入器需要什么信息就直接提供，不拐弯抹角。但是太朴素了，用的人也不多。不过下次我们看到了`$inject`就没有必要焦虑了，它代表的就是注解信息，注入器需要使用它来完成声明参数和实际托管对象之间的关联。而第二种基于数组的方式，实际上就是一个稍微简单一点的写法，并没有什么本质上的改变，使用这种方式算是比较主流的方法。而第三种基于源代码解析的方式就非常酷炫了，大量使用正则表达式给这种方式披上了一层神秘的面纱。也正是因为这个缘故，导致一些开发者觉得这样并不"严格"，才导致了angular提供了一个在bootstrap阶段会读取一个配置项，来决定是否启用这个功能，也就是我们前面见到的`strictDi`标志：

```js
function bootstrap(element, modules, config) {
  // ......
  // 如果config.strictDi为true，那么会禁用基于源代码解析的注解生成方式
  var injector = createInjector(modules, config.strictDi);
  // ......
}
```

其实这种方式不仅仅是不"严格"的问题。众所周知，应用在被部署到生产环境中之前通常还会经过一系列的处理。典型的比如JavaScript源代码的压缩和混淆，这些操作的主要目的分别是减少代码体积从而减少客户端的带宽压力以及增强代码的安全性。而混淆操作由于它会修改函数的参数名称，导致需要对象的名称和被托管对象之间的联系被切断。因此如果你的代码使用了基于源代码解析的方式实现依赖注入并且在使用前还经历了混淆处理，那么你的代码很可能就无法使用了。基于这一点，一般认为使用数组方式来实现依赖注入比较可靠，

但是这种方式相比直接使用`$inject`而言虽然稍微方便了那么一点，但是始终还是不够方便。有没有办法既能够很方便地声明依赖，就像使用基于源码解析的方式那样，而同时也能够克服混淆带来的问题呢？"懒惰"的开发人员们能懒一点是一点，开发出了一些工具来让这个过程自动化起来。

比如下面这款工具[ng-annotate](https://www.npmjs.com/package/ng-annotate)，使用它能够完成如下效果：

```js
// 定义方式如下：需要在function的内部第一行使用"ngInject"
angular.module("MyMod").controller("MyCtrl", function($scope, $timeout) {
    "ngInject";
    ...
});

// 然后通过运行ng-annotate -a 源文件 来得到自动生成的基于数组方式的注入方式：
angular.module("MyMod").controller("MyCtrl", ["$scope", "$timeout", function($scope, $timeout) {
    "ngInject";
    ...
}]);
```

但是这还是不够自动，每次还需要运行一个命令。因此借助gulp等自动化构建工具，出现了更"懒"的[gulp-ng-annotate](https://www.npmjs.com/package/gulp-ng-annotate/)，这样依赖注入所需要的注解信息直接通过定义task的方式自动生成，而这些task还可以和其它的诸如watch等task整合在一起，每次源代码发生变化的时候都会即使更新：

```js
var gulp = require('gulp');
var ngAnnotate = require('gulp-ng-annotate');

gulp.task('default', function () {
    return gulp.src('src/app.js')
        .pipe(ngAnnotate())
        .pipe(gulp.dest('dist'));
});
```

以上就是和依赖注入密切相关的注解，它在angular中的实现方式。

在下一篇文章中，会介绍angular提供给外部使用的`$injector`服务，有了这篇文章和前面数篇文章的铺垫，再理解`$injector`服务就一点也不难了。


