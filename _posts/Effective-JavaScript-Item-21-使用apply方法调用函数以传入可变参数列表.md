---
title: '[Effective JavaScript] Item 21 使用apply方法调用函数以传入可变参数列表'
date: 2014-09-17 13:27:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。
 
## 重点 

下面是一个拥有可变参数列表的方法的典型例子：

```js
average(1, 2, 3); // 2  
average(1); // 1  
average(3, 1, 4, 1, 5, 9, 2, 6, 5); // 4  
average(2, 7, 1, 8, 2, 8, 1, 8); // 4.625  
```

而以下则是一个只接受一个数组作为参数的例子：

```js
averageOfArray([1, 2, 3]); // 2  
averageOfArray([1]); // 1  
averageOfArray([3, 1, 4, 1, 5, 9, 2, 6, 5]); // 4  
averageOfArray([2, 7, 1, 8, 2, 8, 1, 8]); // 4.625 
```

<!-- More -->

毫无疑问，拥有可变参数列表的方法更加简洁和灵活，它可以处理任意多的参数。但是当参数是一个数组时，如何调用拥有可变参数列表的方法呢？

答案是使用内置的apply方法，这个方法和call方法十分类似，除了apply方法是接受一个数组作为参数，然后会将该数组中的每个对象当成单独的参数进行调用。同时，apply方法的第一个参数同call方法的第一个参数意义一样，用来指定this的指向。所以，结合apply方法，就可以处理数组类型的参数了：

```js
var scores = getAllScores();  
average.apply(null, scores);  
```

假设scores是一个拥有三个元素的数组，那么上述调用实际上就是：

```js
average(scores[0], scores[1], scores[2]);  
```

另一个例子是将apply运用到依赖arguments变量的方法中，关于arguments变量的意义和用法，参见Item 22。

```js
var buffer = {  
    state: [],  
    append: function() {  
        for (var i = 0, n = arguments.length; i < n; i++) {  
            this.state.push(arguments[i]);  
        }  
    }  
};  
```

append方法可以使用任意多的参数进行调用，因为它在实现中依赖了arguments变量：

```js
buffer.append("Hello, ");  
buffer.append(firstName, " ", lastName, "!");  
buffer.append(newline);  
```

使用apply方法后，可以这样调用：

```js
buffer.append.apply(buffer, getInputStrings());  
```

需要注意的是，在调用apply的时候，传入了buffer对象作为this的指向，这是因为在append的实现中依赖了this变量，需要显式传入该依赖才能确保修改发生在了正确的对象上。
 
## 总结

1. 使用apply方法来将数组类型的参数传入到接受可变参数列表的方法中
2. 使用apply方法的第一个参数来指定this的指向