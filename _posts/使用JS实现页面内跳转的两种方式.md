---
title: 使用JS实现页面内跳转的两种方式
date: 2015-03-14 12:03:00
categories: ['工具, 库与框架', jQuery]
tags: [jQuery, JavaScript]
---

## 第一种方式是直接使用锚点配合链接标签

```html
<h2 id="h2-anchor">Scroll to here</h2>  
  
<a href="#h2-anchor">Jump to H2</a>  
```

现在大多数实现都采用该种方式。但是这种方式没有动画效果，跳转是直接发生的。

<!-- More -->

## 第二种方式使用jQuery中的animate方法实现：

```js
var target= $('#h2-anchor').offset().top;  
  
$('body').animate({scrollTop:target}, 300);  
```

这种方式跳转的过程更自然。
