---
title: '[Effective Java Distilled] Item 2 当构造方法中有多个参数时，考虑建造者模式'
date: 2013-01-18 14:21:00
categories: [编程语言, Java]
tags: [Java, Effective Java, 设计模式, 最佳实践]
---

### 关于Effective Java Distilled

《Effective Java》这本书我断断续续的读了近两遍，里面的内容挺有深度，对提高工程代码质量也非常有帮助。我打算慢慢的整理出来一个系列，之所以命名为Effective Java Distilled，也是想将本书的精华尽可能的整理出来，方便复习查阅使用，毕竟自己的记忆力也很有限，很多东西经常忘记，一言以蔽之，就是对这本书的一些读书笔记吧。文章中的内容肯定是忠于原文的，对于某些Items，可能会添加一些内容，添加的内容我都会标明。同时，也希望本系列的文章能给大家带来一些帮助。如有疑问或者建议，烦请留言，谢谢大家。

## 本文提纲

- 参数过多对静态工厂方法和构造方法的影响
- JavaBean模式及其弊端
- 建造者模式的运用
- 泛化建造者模式

<!-- More -->

实际工程中往往有一些实体类，这些实体类通常都会拥有数量不等的域用来保存各种相关状态。有一些状态必须有值，而有一些则是可选状态。对于这类实体的实例化，通常都通过一个包含了所有必选状态的构造方法来完成，同时对其他可选状态提供一些setters方法，用于在初始化后进行可选状态的设置。这种模式已经逐渐成为一种约定，在很多框架中都会被使用。比如Hibernate这种ORM框架，从数据库表中获取记录时，将记录通过这种形式转换成Java对象。
 
一旦习惯这种方式，其实也不会觉得此方式有什么不合适的地方。但是还需要考虑没有框架支持的情况，如果需要通过自己编写代码来完成很多个对象的实例化，以上这种构造方法结合setters的初始化方式无疑还有很大的提升空间。

## 参数过多对静态工厂方法和构造方法的影响

首先，关于静态工厂方法的概念，可以参见Item 1中的内容。
 
参数过多，特别是可选参数过多的情况，对静态工厂方法和构造方法都有不利影响。

比如现在有5个参数，其中a，b是必选参数，c，d，e是可选参数，那么为了创建此对象，在不考虑使用setters的情况下，需要的构造方法(静态工厂方法)的数量为：1 + 3 + 3 + 1 = 8个，具体如下：

- 只带有必选参数的数量：1，即constructor(a, b)
- 带有一个可选参数的数量：3，即constructor(a, b, c) constructor(a, b, d) 和 constructor(a, b, e)
- 带有两个可选参数的数量：3，即constructor(a, b, c, d) constructor(a, b, c, e) 和 constructor(a,b, d, e)
- 带有全部可选参数的数量：1，即constructor(a, b, c, d, e)

### Telescoping Constructor模式

以上这种声明构造方法(或者静态工厂方法)的模式被称为Telescoping  Constructor模式，即先带有必要参数，然后逐渐增加可选参数的数量，直到拥有全部参数。

这种模式能够起作用，但是弊端也不少，首当其冲的就是**构造方法规模过大**。当仅有5个参数，其中3个可选的情况下，就需要要多达8个构造方法(或者静态工厂方法)，那么当可选参数的数量继续增加时，需要的构造方法的数量会呈现爆发式增长的趋势。另外，**当可选参数的类型都相同时，容易将参数的位置弄错**。而这种错误可谓是防不胜防，因为类型一样，编译的时候不会出现错误，而在运行时则会出现各种莫名其妙的问题。

## JavaBean模式及其弊端

主要思想就是只提供一个无参构造方法，然后通过setters方法来设置必要的域和可选的域。

它的主要弊端包括：

1. **代码冗长，当需要设置的域很多时，更加明显**

	这是因为每一个field都需要显式调用它的setter方法。
	
2. **在并发环境中可能出现的不一致性**

	正是由于每个field都需要调用setter方法来设置其值，所以在不采用同步机制的情况下，无法保证对所有的fields的设置为原子操作，当然，可以通过各种同步机制来保证设置操作的原子性，但是这又会牺牲一部分性能。
	
3. **无法将该实体类声明为不可变类(Immutable Class)**

	这是因为在类中引入了setters，这违背了不可变类的基本要求，即在对象创建之后，对象中的任何值都不再可变。可变类在单线程环境中几乎没有风险，但是在并发环境下，可能存在各种风险。这个在后续并发相关的Items中会讨论。
	
4. **setters方法可能破坏不变性约束(Invariants)**

	如果实体类中的几个域之间有不变性约束关系，使用setters方法来赋值的时候，可能会造成该约束破坏，从而给程序运行带来隐患。不变性约束的遵循在并发环境中十分重要，此概念在后序并发相关的Items中会讨论到。
	
## 建造者模式的运用

无论是Telescoping Constructor模式还是JavaBean模式，使用它们的原因很大程度上是因为Java仅仅支持定位参数(Positional Parameters)，而不支持命名参数(Named Parameters)。在其他语言，比如Python中，这两种模式都被支持，典型的Python方法签名：def method(*args, **kwargs) 非常明显的体现了这一点，其中args代表了位置参数，而kwargs则是命名参数。很可惜，Java没有内置的语言特性能够支持命名参数，但是通过使用建造者模式，可以模仿这一行为：

直接引用Effective Java中关于营养成分的例子：

```java
// Builder Pattern  
public class NutritionFacts {  
    private final int servingSize;  
    private final int servings;  
    private final int calories;  
    private final int fat;  
    private final int sodium;  
    private final int carbohydrate;  
    public static class Builder {  
        // Required parameters  
        private final int servingSize;  
        private final int servings;  
        // Optional parameters - initialized to default values  
        private int calories = 0;  
        private int fat = 0;  
        private int carbohydrate = 0;  
        private int sodium = 0;  
        public Builder(int servingSize, int servings) {  
            this.servingSize = servingSize;  
            this.servings = servings;  
        }  
        public Builder calories(int val)  
            { calories = val; return this; }  
        public Builder fat(int val)  
            { fat = val; return this; }  
        public Builder carbohydrate(int val)  
            { carbohydrate = val; return this; }  
        public Builder sodium(int val)  
            { sodium = val; return this; }  
        public NutritionFacts build() {  
            return new NutritionFacts(this);  
        }  
    }  
    private NutritionFacts(Builder builder) {  
        servingSize = builder.servingSize;  
        servings = builder.servings;  
        calories = builder.calories;  
        fat = builder.fat;  
        sodium = builder.sodium;  
        carbohydrate = builder.carbohydrate;  
    }  
}  
```

相应构造过程的代码为：

```java
NutritionFactscocaCola = new NutritionFacts.Builder(240, 8).  
    calories(100).sodium(35).carbohydrate(27).build();
```

以上使用链式赋值，看上去比使用setters更加自然，同时代码冗余也更少。使用链式赋值的方式也神似使用命名参数进行赋值的场景，上面的calories，sodium以及carbohydrate等可选参数可以被看做key，后面接的参数则是value。链式操作的关键点就是链中的每一个方法的返回类型都是该类型本身。

### 建造者模式如何克服Telescoping Constructor模式和JavaBean模式的缺点

1. **TC模式中构造方法规模过大的问题**

	显然，使用建造者模式之后，构造方法只有两个，一个是建造者本身的公用的构造方法，它只会带有所有必选的参数；另一个是实体类本身声明为private的构造方法，它仅能通过建造者实例调用。因此，实际公开给外部代码使用的构造方法也只有一个而已。
	
2. **TC模式中参数容易混淆的问题**

	对于必选参数数量有限，而可选参数数量十分多的用例，比如上面的建造者对象中的构造方法只有2个必选参数，因为能够回避在构造方法中声明任何可选参数，所以建造者模式也极大的降低了参数混淆的问题，只有2个参数如果还能混淆，那只能呵呵了……
	
3. **JavaBean模式中构造代码冗长的问题**

	使用链式赋值方式，可以减少构造代码的长度。

4. **JavaBean模式中构造过程非原子性的问题**

	在使用建造者模式来构造对象时，只有当调用了建造者实例的build方法后，对象才能被正常访问，而且当build方法被调用之后，所有的域都会被正常初始化，不存在使用setters时，对象初始化不完整的情况。
	
5. **JavaBean模式中强制性声明类为非可变性的问题**

	从上面的示例代码中，可以看到实体类本身的所有域都被声明为private final，它们没有setters方法，也就是说，一旦对象初始化完毕，它的任何域就不可改变了。这种做法让实体类称为不可变类，从而让该类型在并发环境中能够被更加安全而高效的使用。
	
6. **JavaBean模式中setters方法可能破坏不变性约束的问题**

	这一点，在建造者对象的build方法中可以进行控制。因为build方法负责实例化实体类，而在实例化实体类之前，可以对一些参数进行检查，如果不满足不变性约束，可以根据场景抛出异常，如果没有自定义的异常，一般选择抛出IllegalStateException。
	
## 泛化建造者模式

在Item 1中，提到了静态工厂方法和简单工厂模式有几分相似。这是通过声明静态工厂方法的返回值为父类型来实现的。而利用泛型，则可以让建造者模式和抽象工厂模式进行融合，通过下面这个接口：

```java
// A builder for objects of type T  
public interface Builder<T> {  
    public T build();  
}  
```

为何这里是和抽象工厂模式进行了融合，而不是和简单工厂模式进行融合，这是因为上面定义的接口让构造出来的实际对象能够最多在两个维度上变化。
 
首先，T本身就是一个维度，这里的T只是一个占位符(placeholder)，它能代表任何类型，包括接口类型。如果T是一个接口类型或者某个类层次上的父类的话，那么就开启了第二个维度，因为它本身又代表了一系列的对象。这样一来，后面的事情就类似简单工厂模式了。同样地，如果T类型实际上是一个具体的类，比如上面提到的NutritionFacts，那么抽象工厂模式就退化成了简单工厂模式。

就上面的例子而言，对Builder的声明应该改成：

```java
public class NutritionFacts {
	// ...
	public static class NutritionFactsBuilderimplementsBuilder<NutritionFacts> {
		// ...
	}
}
```

引用《Effective Java》中关于Item 2的总结：
 
> 当构造方法或者静态工厂方法中的参数过多的时候，尤其是可选参数很多时，考虑使用建造者模式吧。它比Telescoping Constructor模式更易读，更简练。同时也比JavaBean模式更加安全。



