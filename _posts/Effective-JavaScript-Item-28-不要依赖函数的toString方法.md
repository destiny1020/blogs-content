---
title: '[Effective JavaScript] Item 28 不要依赖函数的toString方法'
date: 2014-09-25 09:56:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。
 
在JavaScript中，函数对象上存在一个toString方法，它能够方便地将函数的源代码转换返回成一个字符串对象。

```js
(function(x) {  
    return x + 1;  
}).toString(); // "function (x) {\n return x + 1;\n}"  
```

toString方法不仅仅会让一些黑客找到攻击的方法，而且该方法也存在严重的限制。

<!-- More -->
 
首先，toString方法的实现方式并没有被ECMAScript规范化，因此各种JavaScript的执行引擎中的toString的实现方式也许会不一致。
 
其次，当toString能够返回函数源代码并且函数本身完全以JavaScript实现时，源代码才会被正确的返回。比如，在以下的函数调用了，使用了bind方法得到了一个新的函数对象(关于bind的使用方式，可以参考Item 25, 26)：

```js
(function(x) {  
    return x + 1;  
}).bind(16).toString(); // "function (x) {\n [native code]\n}" 
```

可以发现，返回的字符串中有一段是[nativecode]，这是因为在很多JavaScript执行环境中，bind方法都是使用其他编程语言如C++来实现的。因此这里看到native code实际上就是代表着一段编译后的C++源码。
 
最后，toString方法返回的源代码体现不了传入参数的值：

```js
(function(x) {  
    return function(y) {  
        return x + y;  
    }  
})(42).toString(); // "function (y) {\n return x + y;\n}"  
```

上述代码中传入的参数42在返回的函数源代码中并没有被体现出来。
 
因为以上的这些限制，使toString方法很难正确和可靠地被使用。在实际应用中，应该尽量避免使用它。
 
## 总结

1. ECMAScript标准并没有对函数的toString的实现方式作出规范。
2. 因为toString在各种平台上存在着不一致的行为，尽量不要使用它。
