---
title: '[Effective JavaScript] Item 20 使用call方法来绑定this变量'
date: 2014-09-16 09:54:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

通常而言，一个函数中this的指向和该函数的调用类型相关，比如当函数直接作为函数被调用时，this一般指向的是全局对象(StrictMode时指向undefined)；当函数作为方法被调用时(即x.method()这种形式)，this指向的是x；当函数作为构造方法被调用时，this指向的是一个新创建的对象。
 
但是在一些场合，需要指定this的指向，比如下面的代码需要将this指向一个对象obj，一个简单的办法如下：

```js
obj.temporary = f; // what if obj.temporary already existed?  
var result = obj.temporary(arg1, arg2, arg3);  
delete obj.temporary; // what if obj.temporary already existed?  
```

<!-- More -->

但是上述代码假设obj上没有属性叫做temporary，在实际代码中，作出这样的假设显然是不太好的。而且，在某些场合，对象也许会被设置成为不能扩展，也就是不能为它们添加新的属性。而且对不由你创建的对象添加属性也不是一个好的行为，在Item 42中会详细讨论这一点。
 
幸运的是，function提供了内置的call方法来处理这一问题：

```js
f.call(obj, arg1, arg2, arg3);  
```

call方法的第一个参数设置的就是this的指向。
 
使用call方法来调用方法的好处之一：
即使某个对象并没有该方法，也可以通过设置this的指向完成对于该方法的调用，比如：

```js
var hasOwnProperty = {}.hasOwnProperty;  
dict.foo = 1;  
delete dict.hasOwnProperty;  
hasOwnProperty.call(dict, "foo"); // true  
hasOwnProperty.call(dict, "hasOwnProperty"); // false  
```

上述代码首先通过一个空对象取到了hasOwnProperty方法，然后使用call方法来判断某个dictionary对象上是否存在某个key。
 
call方法在定义高阶函数的时候也有用处。在使用高阶函数时一个常见的需求就是提供this的指向，比如：

```js
var table = {  
    entries: [],  
    addEntry: function(key, value) {  
        this.entries.push({ key: key, value: value });  
    },  
    forEach: function(f, thisArg) {  
        var entries = this.entries;  
        for (var i = 0, n = entries.length; i < n; i++) {  
            var entry = entries[i];  
            f.call(thisArg, entry.key, entry.value, i);  
        }  
    }  
};  
```

上述forEach方法定义了两个参数：

- f：一个回调函数
- thisArg：this的具体指向
 
一个例子是将table1中的所有键值对全部拷贝到table2中：

```js
table1.forEach(table2.addEntry, table2);  
```

注意到以上的代码在执行回调函数的时候，传入了除this指向外的另外三个参数，分别是key，value以及index，然后在addEntry方法中并没有定义index参数，这种行为在JavaScript中是合法的，多余的参数会被自动忽略掉，当然如果需要访问并未定义的参数，也可以通过arguments这一变量进行获取。
 
## 总结

1. 使用call方法来在调用函数的时候确定this的指向
2. 使用call方法来调用对象上并不存在的方法
3. 使用call方法来定义高阶函数，让调用者可以指定回调函数中this的指向

