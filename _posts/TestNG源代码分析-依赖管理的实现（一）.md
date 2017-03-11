---
title: TestNG源代码分析 --- 依赖管理的实现（一）
date: 2012-06-04 19:56:00
categories: [源码分析, TestNG]
tags: [源码分析, TestNG, 依赖管理]
---

最近看了一些TestNG的源代码，觉得这个测试框架的功能其实满强大的，里面的功能点很多，以后有机会慢慢分析一下它们的实现方法，今天主要介绍一下它如何实现方法之间的依赖关系。

## 背景知识

想必大家都知道拓扑排序吧，拓扑排序最经典的应用场景就是对于Jobs/Tasks的规划，即对于存在前后依赖关系的任务如何安排一个计划来执行它们。
相关的资料，可以参考[维基百科](http://en.wikipedia.org/wiki/Topological_sorting)，里面介绍的比较详细。

<!-- more -->

那么反映到我们的测试场景中，有一些测试方法之间是存在依赖关系的，比如在对一个Serverside的class进行测试的时候：

```java
@Test  
public void serverStartedOk(){}  

@Test(dependsOnMethods={ "serverStartedOk" })  
public void method1() {} 
```

显然，method1是依赖于serverStartedOK这个方法的，即在被依赖的方法成功执行之后，method1才会执行。
 
当然以上的代码存在一个小问题：

它的可维护性不太好，在method1的@Test注解的dependsOnMethods属性中，直接将依赖方法的名字以字符串的形式传入，这显然是不太合适的。所以，除了这种dependsOnMethods的依赖方式，TestNG还支持了另外一种对于依赖的声明方式，即dependsOnGroups：

```java
@Test(groups = { "init" })  
public void serverStartedOk(){}  

@Test(groups = { "init" })  
public void initEnvironment(){}  

@Test(dependsOnGroups = { "init.*})  
public void method1() {}  
```

以上代码来自：[官方文档](http://testng.org/doc/documentation-main.html#dependent-methods)

首先将初始化方法声明属于init组，然后在method1的@Test的dependsOnGroups属性中将该组的名字传入，如代码中所示，这里指定依赖的groups还能够使用正则表达式，注意这里是正则表达式而不是通配符，因此任意字符串序列使用 .* 而不是 *来进行匹配。

## 实现原理

言归正传，如何在成堆成堆的代码中快速定位到对所有待测试方法进行拓扑排序的实现位置呢？

一开始我也是大海捞针式的进行查找，虽然最后找到了，但是也是花了很多时间，绕了不少弯路。后来在进行试验的时候发现了一个更加快速的方法，即故意制造一个循环依赖，然后看stacktrace：

![](http://my.csdn.net/uploads/201206/05/1338894227_6905.png)

很明显的，Graph类中的topologicalSort方法就是负责拓扑排序的实现了！**(以下的ppp方法是在开启verbose模式后的输出)**

```java
public void topologicalSort() {  
   ppp("================ SORTING");  
   m_strictlySortedNodes=Lists.newArrayList();  // 最后的结果放到这个集合中  
    if (null ==m_independentNodes) {  
     m_independentNodes =Maps.newHashMap();  
    }  
    //Clone the listof nodes but only keep those that are  
    // notindependent.   向nodes2集合中添加非独立的节点，即添加存在依赖关系的节点  
    //  
    List<Node<T>>nodes2 =Lists.newArrayList();  
    for (Node<T> n :getNodes()) {  
      if (!isIndependent(n.getObject())){    // 判断该节点是否独立，如果不独立的话，添加到nodes2中  
       ppp("ADDING FOR SORT: " +n.getObject());  
        nodes2.add(n.clone()); //使用的是clone方法来进行对象的复制，一般不推荐使用clone方法，参见Effective Java Item 11  
      }  
      else {  
       ppp("SKIPPING INDEPENDENT NODE" + n);  
      }  
    }  
    // Sort thenodesalphabetically to make sure that methods of the same class  
    // get run close to eachother as much aspossible  
    // 将非独立的节点集合排序，为了让属于同类中的方法在集合中的位置近一些，从而在调用的顺序上能够相邻一些  
    Collections.sort(nodes2);  
    // Sort  
    while (!nodes2.isEmpty()) {  
      // Find all the nodes that don't have anypredecessors, add  
      // them to the result and mark them forremoval  
      //  从nodes2集合中找到没有前驱节点的节点  
      Node<T> node =findNodeWithNoPredecessors(nodes2);  
      if (null == node) {   // 如果没有找到节点，那么创建一个Tarjan对象来得到一个cycle  
        List<T> cycle =newTarjan<T>(this,nodes2.get(0).getObject()).getCycle(); // 这里实现了Tarjan算法，用来得到环的路径信息  
        StringBuffer sb = new StringBuffer();  //在非并发环境中应该尽量使用StringBuilder  
       sb.append("The following methodshave cyclic dependencies:\n");  
       for (T m : cycle) {  
          sb.append(m).append("\n");  
        }  
       throw newTestNGException(sb.toString());  
      }  
      else {   //如果找到了，将这个没有任何前驱节点的节点放到结果结合中，然后从nodes2集合中删除该节点  
       m_strictlySortedNodes.add(node.getObject());  
       removeFromNodes(nodes2, node);  
      }  
    }  
    ppp("===============DONESORTING");  
    if (m_verbose) {  
     dumpSortedNodes();  
    }  
  }  
```

上面的代码中有几个关键地方：

1. 创建非独立节点集合，即存在依赖关系的方法的集合，用于后续的拓扑排序
2. 预排序上一步得到的集合，使用Collections工具类中的sort方法
3. 进行拓扑排序，如果发现有循环依赖，马上抛出异常

### 第一步：

关键方法：`isIndependent`

```java
public booleanisIndependent(Tobject) {  
  return m_independentNodes.containsKey(object);   
}  
```

该集合中显然只存放了独立节点的refs。

集合的创建过程和上层调用相关，这里暂时略过，只用将它想成是存放独立节点，即不存在任何依赖关系的节点就行了。

### 第二步：

```java
// Sort the nodesalphabetically to makesure that methods of the same class  
// get run close to eachother as much aspossible  
Collections.sort(nodes2);  
```

使用了工具类中的排序方法，而且是对一个集合排序，没有指定任何的Comparator对象，那么显然的，每个node定义了自己的compareTo方法，该方法定义在：

`public static class Node<T> implements Comparable<Node<T>>` 中：

```java
@Override  
public intcompareTo(Node<T> o) {  
  return getObject().toString().compareTo(o.getObject().toString());  
}  
```

相应的toString方法：

```java
@Override  
public String toString(){  
  StringBuffer sb = newStringBuffer("[Node:" +m_object); // 这里首先打印了类型T  
 sb.append(" pred:");  
  for (T o :m_predecessors.values()) {  
    sb.append("" +o.toString());  // 然后加上所有的前驱结点T  
  }  
 sb.append("]");  
  String result= sb.toString();  
  return result;  
}    
```

所以根据上面的toString方法，在m_object相同的成分更多时，排序的结果也就越靠近
这里的m_object的实际类型肯定是方法对象了，那么放在字符串的连接操作符后面，即连接的是toString的返回值。

### 第三步：

`Node<T>node =findNodeWithNoPredecessors(nodes2);`

相应的方法：

```java
private Node<T> findNodeWithNoPredecessors(List<Node<T>> nodes) {  
  for (Node<T> n :nodes) {  
    if (!n.hasPredecessors()) {  
     return n;  
    }  
  }  
  return null;  
}  
```

直接在预排序之后的非独立节点上进行操作：

找到第一个没有前驱结点的node并返回！

Node类中：

```java
public boolean hasPredecessors() {  
  return m_predecessors.size() > 0;  
} 
```

如果对照拓扑排序实现的伪代码，可能看的更清楚(文字部分是TestNG中对应的实现方式)：

```
L ← Empty list that will contain the sorted elements  
```

在方法的开头创建的m_strictlySortedNodeslist就是存放结果的集合

```
S ← Set of all nodes with no incoming edges 
```

实现中并没有独立的创建这么一个入度为0的节点的集合，而是动态的从所有的存在有依赖关系的节点中按需取出一个，也就是说，和后面的while循环一起实现了

```
while S is non-empty do
    remove a node n from S
    insert n into L
    foreach node m with an edge e from nto m do
       remove edge e from thegraph
        ifm has no other incoming edges then
            insert m into S
```

从S集合中remove node通过findNodeWithNoPredecessors方法实现
而remove edge的操作，则是通过removeFromNodes方法实现

```
if graph has edges then
    return error (graph hasat least onecycle)
```

这一步也被实现到了上面的while循环中，这样的好处是，如果发现了有cycle存在，马上就会抛出异常，而不会像这里的伪代码一样，直到while退出了才会发现问题

```
else 
    return L (a topologicallysortedorder)
```

最后的m_strictlySortedNodes即是拓扑排序的结果。

## 结语

以上就是Graph类的大致功能了，当然，这是在假设了很多前提的条件下分析的结果。留下了一些可以继续深究的问题：

- Graph对象中的一些字段是怎么被初始化的？在使用Graph对象的topologicalSort方法的时候，需要用到这些字段，比如m_nodes以及m_independentNodes这两个集合，它们分别存放的是所有的节点的引用以及所有独立节点的引用。
- Graph对象是如何使用的，即方法调用栈的上层是如何调用Graph中的topologicalSort方法的。
关于环路检测算法的实现，用于在发现循环依赖的时候，检测出具体的循环依赖路径。
 
这些问题在下篇文章中会进行介绍！



