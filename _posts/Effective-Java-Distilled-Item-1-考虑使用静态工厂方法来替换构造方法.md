---
title: '[Effective Java Distilled] Item 1 考虑使用静态工厂方法来替换构造方法'
date: 2013-01-17 15:26:00
categories: [编程语言, Java]
tags: [Java, Effective Java, 设计模式, 最佳实践]
---

## 关于Effective Java Distilled

《Effective Java》这本书我断断续续的读了近两遍，里面的内容挺有深度，对提高工程代码质量也非常有帮助。我打算慢慢的整理出来一个系列，之所以命名为Effective Java Distilled，也是想将本书的精华尽可能的整理出来，方便复习查阅使用，毕竟自己的记忆力也很有限，很多东西经常忘记，一言以蔽之，就是对这本书的一些读书笔记吧。文章中的内容肯定是忠于原文的，对于某些Items，可能会添加一些内容，添加的内容我都会标明。同时，也希望本系列的文章能给大家带来一些帮助。如有疑问或者建议，烦请留言，谢谢大家。

## 本文提纲

- 静态工厂方法是什么
- 静态工厂方法的声明方式
- 静态工厂方法的优势
- 静态工厂方法的劣势
- 静态工厂方法的争议

<!-- More -->

## 静态工厂方法是什么

首先，不要把静态工厂方法和设计模式中的几种工厂模式相混淆了。两者没有必然的关联，**静态工厂方法只是一个静态的，用来返回当前类型(或者其子类型)的实例的一个方法**。而工厂模式则涉及到更多的概念，相关概念请参考设计模式中的相关文章。
 
当然，仔细想想，它们之间也存在一些共同点的，它们都是为了更加灵活的创建对象。而且当静态工厂方法声明的返回类型是接口类型的时候，该静态工厂方法的作用就神似简单工厂模式这一基础的创建类设计模式了。

## 静态工厂方法的声明方式

为什么要讨论声明方式，因为静态工厂方法的声明方式和其他静态方法的声明方式几乎没有什么区别。所以，要识别或者定义一个静态工厂方法，更多的需要人为的设置一些约定。主要体现在命名方式上，几种常见的命名方式如下：

- `valueOf`
- `of`
- `getInstance`
- `newInstance`
- `getType`
- `newType`
- ......
 
以上的方法中，getInstance的出镜率很高，在单例类中经常会定义一个getInstance方法用来返回该类型的唯一实例。而valueOf方法和of方法则往往用来解析传入的参数，然后根据它来返回对应的实例。这两个方法常用在多例类中，所谓多例类，就是诸如枚举类型等实例个数不唯一，但是数量有限的类。最常见的例子比如Boolean类，其中定义了一个valueOf方法，接受字符串类型的true和false，然后返回Boolean.True或者Boolean.False。newInstance在反射API中会用到，比如通过调用类的class对象上的newInstance方法来得到一个该类的实例。
 
除了命名方式外，方法的访问修饰符可以根据需要进行设置，这一点和普通方法一致。一般情况下，设置为public即可，因为静态工厂方法的主要用意也是被调用以获取类的实例。
 
另外，既然静态工厂方法是一个普通方法，那么它就必须有返回值。通常而言，这个返回值的声明类型应该是此静态工厂方法所在类的类型。

## 静态工厂方法的优势

当然，这里所说的优势，通常都是和传统构造函数相比得到的。

### 静态工厂方法拥有名字

静态工厂方法拥有自己的名字，而构造函数则没有。这也就意味着，同一组参数列表，只能有唯一的一个构造函数(通过改变参数列表中参数的顺序，可以有多个构造函数，但是很明显，这不是一个好主意)。相比之下，静态工厂方法则没有这个限制，对同一组参数，可以有任意多的静态工厂方法。因此，这种情况下使用静态工厂方法来进行对象实例化更加灵活，可读性也更好。

### 对于对象实例化的更多控制

对于构造函数而言，一旦它被调用，那么就肯定会生成一个新的实例。而调用静态工厂方法则不一定，它只需要返回一个对象的实例就够了，至于这个返回的对象是如何得到的，它并不关注。因此，静态工厂方法对于对象的实例化就有了更多的控制权。它能够利用预先创建好的不可变对象(Immutable Instance)，也能够从缓存中获取某个经常被请求的对象。从而大幅度的提高性能。

### 返回对象的类型更加泛化

对于构造函数而言，它只能返回当前类的实例。而静态工厂方法的实际返回类型可以是声明返回类型的任何子类型。比如，声明返回Base类型的静态工厂方法，实际上它能够返回Sub类型，只要Sub是Base的子类型。因为子类型的概念不仅限于继承层次，还可以适用在接口和其实现类之间，因此这种泛化的关系也非常契合“面向接口编程”这一指导原则。

### 静态工厂方法能够使用类型推导

要实例化一个泛型类，有可能要写非常冗长的初始化语句，例如：

`Map<String,List<String>> m =newHashMap<String, List<String>>();`

其实这还算是好的，如果你需要使用一个嵌套了若干层的HashMap，那么写对它的实例化代码还真不那么轻松，这个时候，可以考虑使用静态工厂方法进行一次封装，借助它的类型推导特性：

```JAVA
publicstatic <K, V> HashMap<K, V> newInstance() {
	return new HashMap<K, V>();
}
```

然后，通过下面的代码来创建：

`Map<String,List<String>> m = HashMap.newInstance();`

当然，对于在这个场景下使用类型推导，究竟是好是坏，好的方面是它确实减少了代码量，但是坏的方面也很明显，它的可读性没有那么强，容易让人迷惑。
 
关于类型推导，在《Java编程思想，第四版》的第15章的15.4.1小节有介绍。

## 静态工厂方法的劣势

### 静态工厂方法和普通静态方法没有区分度

如果不靠命名规范，那么区分静态工厂方法和普通静态方法还是不那么容易的，至少不能一眼就看出来。所以，在目前缺少JavaDoc和内置语言特性支持的情况下，使用靠谱的命名规范是十分重要的，关于命名的建议，在前文也有描述。

## 静态工厂方法的争议

如果一个类型中，**仅**提供静态工厂方法而不提供public或者protected的构造函数，会使该类型不可被继承。继承依赖于构造函数，这是因为在使用子类型的构造函数的时候，会调用父类型的构造函数。
 
注意前面的“仅”字，一般而言，上面这种情况是不会发生的，除非是特意为之，比如Java Collection Framework中的Collections类，其中就只含有一些静态工厂方法，该类被设计为不能被实例化的类，那么它就自然和继承无关了，关于这一点，可以参考Item 4。

而其中包含的那些静态工厂方法的声明方式和实现方式，和上面的争议，实际上是不同的问题，不要混淆了。这里也简要的对它们进行讨论：
这些静态工厂方法创建的对象都是一些包装对象(Wrapper Object)，比如synchronizedList方法，它接受一个List对象并返回一个List对象，这个返回的List对象的具体实现完全被隐藏了，具体实现类的可见性都是package级别，也就是包外的代码是无法访问，无法继承的，摘要部分代码如下所示：

```java
public static <T> List<T> synchronizedList(List<T> list) {  
	return (list instanceof RandomAccess ?  
            new SynchronizedRandomAccessList<T>(list) :  
            new SynchronizedList<T>(list));  
}  
```

以上的代码通过判断该List实现是否可以顺序访问，来决定使用哪一种具体实现类型。
所谓顺序访问，就是可以使用索引(index)进行类似数组访问的性质。就List接口的实现而言，支持顺序访问的List实现类包括ArrayList，ArrayDeque等，而不支持顺序访问的List接口的实现类包括LinkedList等，也就是我们常说的链表。

下面的代码是当List支持顺序访问时，会实例化的包访问级别的内部静态类。这里使用静态类的原因在于，该类的实例不需要持有其容器类的引用，就此例而言，容器类即为Collections，它本身就是一个不可实例化的类，因此不能持有它的引用也是理所应当的了。关于内部类的种类和使用方式，在后面的Items中会介绍。

```java
public static class SynchronizedRandomAccessList<E> extends SynchronizedList<E> implements RandomAccess {  
  
   SynchronizedRandomAccessList(List<E> list) {  
       super(list);  
   }  
  
	SynchronizedRandomAccessList(List<E> list, Object mutex) {  
       super(list, mutex);  
   }
}
```

注意到这个类有继承了一个叫做SynchronizedList的类，这个类转而又继承自SynchronizedCollection类，即：

**SynchronizedRandomAccessList**-> SynchronizedList -> SynchronizedCollection

这个继承层次很眼熟，Java Collections Framework中的基础实现类也是按照这种思路展开的，即：

ArrayList -> AbstractList -> AbstractCollection

实际上，这也是一种实现策略，在后面介绍组合优于继承的时候，会谈到这种策略的用法和好处。

绕了一大圈，回到静态工厂方法本身的讨论上：
synchronizedList方法的返回类型是List接口，而不是诸如SynchronizedRandomAccessList这种实现类。这是因为该实现类不会提供除List接口中定义的那些方法之外的任何方法，因此暴露该实现类给外部就不存在任何好处，反而还会引起一些不必要的麻烦，带来不必要的复杂性。而这一点，也利用了静态工厂方法的优势：

1. 对类型的实例化有更多的控制 
2. 返回对象的类型更加泛化(使用接口而非具体实现类)

在《Effective Java》中，这个争议实际上被归为静态工厂方法的最主要的一个弊端，但是貌似作者本人也不是很认同这一点，原文中提到了：

> For example, it is impossible to subclass any of the convenience implementation classes in the Collections Framework. Arguably this can be a blessing in disguise, as it encourages programmers to use composition instead of inheritance.
> 
比如，不能实例化JavaCollections Framework中的任何实现类。但是这确是一个暗地里的好处，因为它鼓励我们去实现组合而非继承。
