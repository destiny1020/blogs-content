---
title: '[Effective JavaScript] Item 39 绝不要重用父类型中的属性名'
date: 2014-10-14 10:22:00
categories: [编程语言, JavaScript]
tags: [JavaScript, Effective JavaScript, 设计模式, 最佳实践]
---

本系列作为Effective JavaScript的读书笔记。
 
## 重点 
 
如果需要向Item 38中的Actor对象添加一个ID信息：

```js
function Actor(scene, x, y) {  
    this.scene = scene;  
    this.x = x;  
    this.y = y;  
    this.id = ++Actor.nextID;  
    scene.register(this);  
}  
Actor.nextID = 0;  
```

<!-- More -->

同时，也需要向Actor的子类型Alien中添加ID信息：

```js
function Alien(scene, x, y, direction, speed, strength) {  
    Actor.call(this, scene, x, y);  
    this.direction = direction;  
    this.speed = speed;  
    this.strength = strength;  
    this.damage = 0;  
    this.id = ++Alien.nextID; // conflicts with actor id!  
}  
Alien.nextID = 0; 
```

在Alien的构造函数中，也对id属性进行了赋值。因此，在Alien类型的实例中，id属性永远都是通过Alien.nextID进行赋值的，而不是Actor.nextID。父类型的id属性被子类型的id属性覆盖了。
 
解决方案也很简单，在不同的类型中使用不同的属性名：

```js
function Actor(scene, x, y) {  
    this.scene = scene;  
    this.x = x;  
    this.y = y;  
    this.actorID = ++Actor.nextID; // distinct from alienID  
    scene.register(this);  
}  
Actor.nextID = 0;  
function Alien(scene, x, y, direction, speed, strength) {  
    Actor.call(this, scene, x, y);  
    this.direction = direction;  
    this.speed = speed;  
    this.strength = strength;  
    this.damage = 0;  
    this.alienID = ++Alien.nextID; // distinct from actorID  
}  
Alien.nextID = 0;  
```

## 总结

1. 注意所有父类型中使用的属性名称不要和子类型中的重复。
2. 在子类型中不要重用父类型中已经使用的属性名。