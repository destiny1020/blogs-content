---
title: '[Effective Java Distilled] Item 4 通过私有构造方法来加强化不可实例化的性质'
date: 2013-01-19 23:32:00
categories: [编程语言, Java]
tags: [Java, Effective Java, 设计模式, 最佳实践]
---

## 关于Effective Java Distilled
《Effective Java》这本书我断断续续的读了近两遍，里面的内容挺有深度，对提高工程代码质量也非常有帮助。我打算慢慢的整理出来一个系列，之所以命名为Effective Java Distilled，也是想将本书的精华尽可能的整理出来，方便复习查阅使用，毕竟自己的记忆力也很有限，很多东西经常忘记，一言以蔽之，就是对这本书的一些读书笔记吧。文章中的内容肯定是忠于原文的，对于某些Items，可能会添加一些内容，添加的内容我都会标明。同时，也希望本系列的文章能给大家带来一些帮助。如有疑问或者建议，烦请留言，谢谢大家。

## 本文提纲

- 什么类不需要实例化
- 注意事项

此Item也许是所有Item中内容最少的一个了。但是它也涵盖了我们日常编码中比较常用的几个技巧。在创建工具类的时候，会使用到。

<!-- More -->

## 什么类不需要实例化

一言以蔽之，完全用来提供静态工具方法的类型，不需要被实例化。在JDK中，有几个典型：

- java.util.Arrays
- java.util.Collections
- java.lang.Math

这几个类的声明方式分别为：

```java
public class Arrays {  
    // Suppresses default constructor, ensuring non-instantiability.  
    private Arrays() {  
    }  
     …  
}  
  
public class Collections {  
    // Suppresses default constructor, ensuring non-instantiability.  
    private Collections() {  
    }  
     …  
}  
  
public final class Math {  
    /** 
     * Don't let anyone instantiate this class. 
     */  
    private Math() {}  
     …  
}  
```

以上的代码片段全部摘自JDK源码。可以发现，它们都使用了一个私有的构造方法，该私有构造方法内部没有任何逻辑。这样做是为了告诉编译器不要在编译该类型时为它生成默认的无参公有构造方法，即如果一个类没有提供任何构造方法的话，编译器会为它生成一个默认构造方法。而一旦提供了任何构造方法(任何访问级别都可以)，编译器就不会提供该默认构造方法了。
 
另外注意到以上Math类的声明方式和Collections以及Arrays类有些不一样，Math类的声明使用了final关键字，那么这个关键字有无什么特别的意义呢，《Thinking in Java》中的Reusing Classes一章中的The final keyword小节对final关键字在各个场景下的语义进行了讲解。当final用来修饰class，表示该class不可拥有子类。在被final修饰的类型中，所有的方法都是final的，因为他们不可能被子类重写了。

所以在我们讨论的这个场景中，既然已经为类型提供了唯一的私有构造方法，那么也就意味着该类型不能拥有任何子类，原因在于子类的构造方法需要显式或者隐式的使用父类的非私有构造方法。因此，在这个场景下，使用final或者不使用，产生的效果都是相同的。
 
然而，对于为何Math类的声明被final关键字修饰，而Arrays和Collections没有，我想可能是因为每个人的编码习惯和思维方式不太一样吧……Math类的作者是Joseph D. Darcy，而后两个类的两位作者Joshua Bloch和Neal Gafter，他们合著了一本有趣的《Java Puzzlers》，同时Joshua Bloch本人即是《Effective Java》的作者。

## 注意事项

为了让类型不可被实例化，将该类型声明为抽象类是不正确的方式。因为抽象类的本意是为了被子类继承，而创建子类的实例会调用到抽象类的构造方法，因此抽象类本身还是被间接的实例化了。
 
除了使用私有构造方法之外，还可以在该构造方法中添加一行代码，引用《Effective Java》Item 4中的代码段：

```java
// Noninstantiable utility class  
public class UtilityClass {  
    // Suppress default constructor for noninstantiability  
    private UtilityClass() {  
        throw new AssertionError();  
    }  
    // Remainder omitted  
}  
```

即可以通过抛出一个断言类异常来防止在该类型中无意地调用了私有构造方法。同时，这种方法也能够对反射调用私有构造方法say no。这个技巧在Item3，关于单例类的实现中，也有用到。
