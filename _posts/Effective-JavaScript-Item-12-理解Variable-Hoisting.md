---
title: '[Effective JavaScript] Item 12 理解Variable Hoisting'
date: 2014-09-09 10:55:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。
 
## 重点 

JavaScript中并没有Block Scoping，只有Function Scoping。
因此如果在一个Block中定义了一个变量，那么这个变量相当于是被定义到了这个Block属于的Function中，比如：

```js
function isWinner(player, others) {  
    var highest = 0;  
    for (var i = 0, n = others.length; i < n; i++) {  
        var player = others[i];  
        if (player.score > highest) {  
            highest = player.score;  
        }  
    }  
    return player.score > highest;  
}  
```

<!-- More -->

上面的代码中，在for循环中声明了一个变量player，因为Variable Hoisting的原因，这个变量实际上被声明成了下面这个样子：

```js
function isWinner(player, others) {  
    var player;  
    var highest = 0;  
    for (var i = 0, n = others.length; i < n; i++) {  
        <span style="color:#ff0000;">player = others[i];</span>  
        if (player.score > highest) {  
            highest = player.score;  
        }  
    }  
    return player.score > highest;  
}  
```

因此，传入到function中的player参数被覆盖掉了。程序不能按预期行为运行。
 
在JavaScript中，对于变量的声明实际上包含了两个部分：

1. 声明本身
2. 赋值

JavaScript会通过Variable Hoisting将第一个部分，也就是声明的部分，放到包含变量的function的头部。
 
在下面这个例子中：

```js
function trimSections(header, body, footer) {  
    for (var i = 0, n = header.length; i < n; i++) {  
        header[i] = header[i].trim();  
    }  
    for (var i = 0, n = body.length; i < n; i++) {  
        body[i] = body[i].trim();  
    }  
    for (var i = 0, n = footer.length; i < n; i++) {  
        footer[i] = footer[i].trim();  
    }  
}  
```

因为VariableHoisting的缘故，i和n都会被放到function的开始处，所以实际上只声明了两个变量，在for循环中，会对它们进行赋值，实际的行为是这样的：


```js
function trimSections(header, body, footer) {  
    var i, n;  
    for (i = 0, n = header.length; i < n; i++) {  
        header[i] = header[i].trim();  
    }  
    for (i = 0, n = body.length; i < n; i++) {  
        body[i] = body[i].trim();  
    }  
    for (i = 0, n = footer.length; i < n; i++) {  
        footer[i] = footer[i].trim();  
    }  
}  
```

正因为VariableHoisting的存在，一些程序员偏好将变量定义在function的开始处，相当于完成了一次手动地Hoisting。这样做的目的是为了避免歧义性，让代码更加清晰。
 
但是，有一个特殊情况，在try…catch语句中，catch block中的参数会被绑定到该block中：

```js
function test() {  
    var x = "var", result = [];  
    result.push(x);  
    try {  
        throw "exception";  
    } catch (x) {  
        x = "catch";  
    }  
    result.push(x);  
    return result;  
}  
test(); // ["var", "var"]  
```

## 总结

1. 在Block中声明的变量会被隐式地Hoisted到包含它们的function的开始处。（除了上面提到的catch block）
2. 在一个function中，多次声明一个变量最终只会声明一个变量。
3. 为了避免歧义性，考虑将变量声明在function的开始处。


