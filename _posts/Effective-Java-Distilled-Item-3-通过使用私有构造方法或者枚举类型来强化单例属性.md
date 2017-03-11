---
title: '[Effective Java Distilled] Item 3 通过使用私有构造方法或者枚举类型来强化单例属性'
date: 2013-01-19 11:40:00
categories: [编程语言, Java]
tags: [Java, Effective Java, 设计模式, 最佳实践]
---

## 关于Effective Java Distilled

《Effective Java》这本书我断断续续的读了近两遍，里面的内容挺有深度，对提高工程代码质量也非常有帮助。我打算慢慢的整理出来一个系列，之所以命名为Effective Java Distilled，也是想将本书的精华尽可能的整理出来，方便复习查阅使用，毕竟自己的记忆力也很有限，很多东西经常忘记，一言以蔽之，就是对这本书的一些读书笔记吧。文章中的内容肯定是忠于原文的，对于某些Items，可能会添加一些内容，添加的内容我都会标明。同时，也希望本系列的文章能给大家带来一些帮助。如有疑问或者建议，烦请留言，谢谢大家。

## 本文提纲

- 传统的实现单例模式的几种方式
- Lazy方式实现单例模式
- 使用枚举类型来实现单例模式

需要说明的是，关于第二点，Lazy方式实现单例模式在《Effective Java》原文中没有提到。这里是为了完整性，自行添加的内容，是否有实践意义，还有待商榷，因为使用Lazy实现方式，特别在并发环境中使用它，到底能带来多少好处，确实不好说。其实我个人感觉是，不会带来什么好处。带来的那么一点点性能上以及空间上的好处，实在不足以抵消它引入的复杂性。

<!-- More -->
 
另外，代码都是直接引用原文的代码片段或者在其基础之上改写而得到。

## 传统的实现单例模式的几种方式

主要是两种方式：

1. 私有构造方法 + 公共static域 + Eager实现
2. 私有构造方法 + 静态工厂方法 + Eager实现

关于Lazy方式，请参见下个section。

可以发现，两种方式的唯一区别就在于一个是使用的公共static域来暴露单例对象，而另一个是通过静态工厂方法来进行暴露。使用静态工厂方法的好处主要有两点：

1. 通过静态工厂方法引入了一层间接性，在方法中可以更改实现，来满足需求的变更，比如单例是全局单例(即在整个运行时都只有这一个实例)还是线程单例(即每个线程都能拥有该对象的唯一实例)。
2. 符合泛型思想，目的是要让单例对象能够跨越多种类型存在，具体请参见后续泛型中的相关Items。
 
但是这两个优点带来的好处往往**不那么重要**，相比之下，使用公共静态域还更简洁一些。原文中使用的是relevant，有一个生僻义是"有重大作用的"，这里要吐槽一下EJ中文版的翻译，翻译的是"相关的"……放在上下文中，明显说不通的。

原文如下：
> Often neither of these advantages is **relevant**, and the final-field approach is simpler.

同时，以上两个方法都面临的同样的问题，即私有构造方法并不能保证它绝对不会被外部调用。一个具有相关优先级的客户端可以借助AccessibleObject.setAccessible，从而通过反射的方式来调用私有构造方法来获取更多的实例。因此，针对这种情况，可以在构造方法中添加判断，如下所示：

```java
// Singleton with static factory  
public class Elvis {  
    private static final Elvis INSTANCE = new Elvis();  
    private Elvis() {  
        if (INSTANCE != null) {  
            throw new IllegalStateException("单例类不能创建第二个实例")  
        }  
    }  
    public static Elvis getInstance() { return INSTANCE; }  
}  
```

另外，对于可序列化的单例类，还需要自定义一个readResolve方法，用来自定义在反序列化时返回的对象，不这样做的话，每次在进行反序列化时，都会生成一个新的实例。

```java
// readResolve method to preserve singleton property  
private Object readResolve() {  
    // Return the one true Elvis and let the garbage collector  
    // take care of the Elvis impersonator.  
    return INSTANCE;  
}  
```

## Lazy方式实现单例模式

在上面介绍的两种实现方法中，采用的策略都是Eager方式，即在定义INSTANCE的同时就会创建该对象。当然除了Eager策略之外，还有Lazy策略。即只定义staticINSTANCE，而不马上就对它进行初始化。
 
非并发环境下，Lazy方式的实现思路很简单，在Get单例对象的时候，判断其是否为空，如果是空，那么首先初始化该单例对象，然后返回之。

```java
// Singleton with static factory (Lazy Strategy)  
public class Elvis {  
    private static final Elvis INSTANCE;  
    private Elvis() {  
        if (INSTANCE != null) {  
            throw new IllegalStateException("单例类不能创建第二个实例");  
        }  
    }  
    public static Elvis getInstance() {  
        if(INSTANCE == null) {  
            INSTANCE = new Elvis();  
        }  
        return INSTANCE;  
    }  
}  
```

而在并发环境下，Lazy方式的单例实现就没有那么简单了。最重要的原则是，不能让多个线程创建出多于一个的实例。因此，在创建实例的那一部分，需要进行加锁：

```java
// Singleton with static factory (Lazy Strategy in Concurrent Environment)  
public class Elvis {  
    private static Elvis INSTANCE;  
    private Elvis() {  
        if (INSTANCE != null) {  
            throw new IllegalStateException("单例类不能创建第二个实例");  
        }  
    }  
    public static Elvis getInstance() {  
        if(INSTANCE == null) {  
            synchronized {  
                if(INSTANCE == null) {  
                    INSTANCE = new Elvis();  
                }  
            }  
        }  
        return INSTANCE;  
    }  
}  
```

上面的代码使用了双重检查锁定的技巧，看起来没有什么问题，为了提高性能，将同步代码块压缩到了比较小的范围，但是这个方法有很多争议，通常的建议是不要使用它。具体可以参考这个[链接](http://www.ibm.com/developerworks/cn/java/j-dcl.html)

链接中的这篇文章比较老。但是对这个问题本身有比较详细的分析，还是有一些启发意义。原文比较长，这里就只谈谈该文章中的关键部分：

双重检查锁定失败的原因不归咎与JVM的实现，而是归咎于Java平台的内存模型。内存模型允许“无序写入”是造成失败的一个主要原因。

文章还强调，即使使用volatile关键字也不能达到预期效果，他给出的理由是：大多数的JVM没有正确的实现volatile，因此不能依赖它的行为。这是因为在Java 1.4中，volatile关键字的功能并没有保障，这一点在1.5中已经得到了纠正。

根据《Java Concurrency In Practice》中有关volatile关键字的介绍，volatile是作为轻量级同步机制而存在的，即它能够抑制编译器对代码的重排序，同时还能够让对volatile变量的操作被其它线程可见，即保证了该变量的内存可见性。之所以是轻量级的同步机制，因为完全同步，比如使用了synchronized修饰的代码块或方法往往包含了两个语义：(1)原子性，(2)内存可见性，而volatile仅仅能够保证内存可见性。

另外，维基百科上对这个问题也有很详细的论述，详见[WIKI](http://en.wikipedia.org/wiki/Double-checked_locking)

简要提一下里面比较重要的观点：

它提到了在J2SE 5.0中，volatile关键字能够起作用，还给出了一个代码示例，这里直接引用过来：

```java
// Works with acquire/release semantics for volatile  
// Broken under Java 1.4 and earlier semantics for volatile  
class Foo {  
    private volatile Helper helper = null;  
    public Helper getHelper() {  
        Helper result = helper;  
        if (result == null) {  
            synchronized(this) {  
                result = helper;  
                if (result == null) {  
                    helper = result = new Helper();  
                }  
            }  
        }  
        return result;  
    }  
    // other functions and members...  
}  
```

另外，Wiki中还介绍了一个实现Lazy方式非常好，非常巧妙的方法，同时它也是线程安全的。通过借助内部静态类以及JVM的类加载机制来实现：

```java
// Correct lazy initialization in Java   
@ThreadSafe  
class Foo {  
    private static class HelperHolder {  
       public static Helper helper = new Helper();  
    }  
   
    public static Helper getHelper() {  
        return HelperHolder.helper;  
    }  
}  
```

它利用了内部静态类只有在被引用的时候才会被加载的规律。

这样一来，一旦内部的HelperHolder被引用了，它就会首先被JVM加载，进行该类的静态域的初始化，从而使得Helper这一单例类被初始化。它之所以是线程安全的，也是托了JVM的福，因为JVM对于类的加载这一过程是线程安全的。

## 使用枚举类型来实现单例模式

通过上面的分析，Eager策略比Lazy策略简单的不是一丁半点。Lazy策略看似十分聪明，但是这种聪明究竟能够带来多少好处确实不好说。对于这种情况，可以参考高德纳的名言“过早的优化是万恶之源”。的确，对于这种效果未知的优化，还不如不要优化，它引入了太多的不确定性。如果能够判断系统的瓶颈在于单例类的实现策略，那么再去投入精力优化也不迟(不过通常而言，这里是不会成为系统瓶颈的……)。

对于Eager策略，使用单元素的枚举类型来实现，是更加简单，更加安全的方法：

```java
// Enum singleton - the preferred approach  
public enum Elvis {  
    INSTANCE;  
    public void leaveTheBuilding() { ... }  
}  
```

这种方式帮你避免了通过序列化和反射可能带来的问题。

至于它为何能够屏蔽序列化和反射的攻击，在后续介绍枚举类型的时候，会提到。
 
最后，引用原文中的总结：

> 这种方式虽然还没有得到普及，但是它确实是最好的实现单例模式的方法。


