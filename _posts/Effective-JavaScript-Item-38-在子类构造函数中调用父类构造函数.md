---
title: '[Effective JavaScript] Item 38 在子类构造函数中调用父类构造函数'
date: 2014-10-13 17:17:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。

## 重点
 
在一个游戏或者图形模拟的应用中，都会有场景(Scene)这一概念。在一个场景中会包含一个对象集合，这些对象被称为角色(Actor)。而每个角色根据其类型会有一个图像用来表示，同时场景也需要保存一个底层图形展示对象的引用，被称为上下文(Context)：

```js
function Scene(context, width, height, images) {  
    this.context = context;  
    this.width = width;  
    this.height = height;  
    this.images = images;  
    this.actors = [];  
}  
Scene.prototype.register = function(actor) {  
    this.actors.push(actor);  
};  
Scene.prototype.unregister = function(actor) {  
    var i = this.actors.indexOf(actor);  
    if (i >= 0) {  
        this.actors.splice(i, 1);  
    }  
};  
Scene.prototype.draw = function() {  
    this.context.clearRect(0, 0, this.width, this.height);  
    for (var a = this.actors, i = 0, n = a.length; i < n; i++) {  
        a[i].draw();  
    }  
};  
```

<!-- More -->

场景中所有的角色都继承自一个基类，这个基类用来抽象所有角色具有的属性和方法。比如，每个角色对象都会保存它所在场景的引用，和坐标信息：

```js
function Actor(scene, x, y) {  
    this.scene = scene;  
    this.x = x;  
    this.y = y;  
    scene.register(this);  
}  
```

同样地，在Actor类型的prototype对象上会定义公共的方法：

```js
Actor.prototype.moveTo = function(x, y) {  
    this.x = x;  
    this.y = y;  
    this.scene.draw();  
};  
Actor.prototype.exit = function() {  
    this.scene.unregister(this);  
    this.scene.draw();  
};  
Actor.prototype.draw = function() {  
    var image = this.scene.images[this.type];  
    this.scene.context.drawImage(image, this.x, this.y);  
};  
Actor.prototype.width = function() {  
    return this.scene.images[this.type].width;  
};  
Actor.prototype.height = function() {  
    return this.scene.images[this.type].height;  
};  
```

有了角色基础类，就可以在其之上创建具体类型了。比如当创建一个宇宙飞船(SpaceShip)角色时，可以这样实现：

```js
function SpaceShip(scene, x, y) {  
    Actor.call(this, scene, x, y);  
    this.points = 0;  
}  
```

为了让SpaceShip的实例也能够拥有所有角色应该有的属性，所以在SpaceShip的构造函数体内首先调用了父类(Actor)的构造函数，紧接着会初始化SpaceShip实例自身的属性，比如以上的points。
 
为了让SpaceShip类型确确实实地成为Actor类型的子类型，SpaceShip类型的prototype对象也必须要继承自Actor类型的prototype对象。这可以通过ES5提供的Object.create方法完成(非ES5的实现方式可以参考Item 33)：

```js
SpaceShip.prototype = Object.create(Actor.prototype);  
```

如果SpaceShip的prototype对象是通过调用Actor的构造函数来获得的，那么会出现一系列的问题：

```js
SpaceShip.prototype = new Actor();  
```

在调用Actor构造函数的时候，没法传入合理的参数。因为Actor接受场景对象和坐标信息作为参数，而SpaceShip类型的prototype对象的目的是为了容纳SpaceShip类型中一些公用的属性和方法，显然场景和坐标信息会随着SpaceShip实例的不同而不同，将这些信息放在prototype对象上是不合适的。
 
父类型的构造函数只能在子类型的构造函数中被调用，而子类型的prototype对象是继承自父类型的prototype对象。这一点在创建子类型的prototype时需要注意。
 
一旦完成了子类型prototype对象的创建，就可以在其上设置公用的属性和方法了：

```js
SpaceShip.prototype.type = "spaceShip";  
SpaceShip.prototype.scorePoint = function() {  
    this.points++;  
};  
SpaceShip.prototype.left = function() {  
    this.moveTo(Math.max(this.x - 10, 0), this.y);  
};  
SpaceShip.prototype.right = function() {  
    var maxWidth = this.scene.width - this.width();  
    this.moveTo(Math.min(this.x + 10, maxWidth), this.y);  
};  
```

此时，Actor类型，SpaceShip类型以及它们的prototype对象之间的关系如下：

![](http://img.blog.csdn.net/20141013171706068?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZG1fdmluY2VudA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

## 总结

1. 在子类型的构造函数中调用父类型的构造函数，并显式传入this的指向。
2. 使用Object.create方法创建子类型的prototype对象来避免对父类型构造函数的调用。