---
title: TestNG源代码分析 --- 依赖管理的实现（二）
date: 2012-06-07 13:42:00
categories: [源码分析, TestNG]
tags: [源码分析, TestNG, 依赖管理]
---

在上一篇文章中，留下了一些的问题：

- Graph对象中的一些字段是怎么被初始化的？在使用Graph对象的topologicalSort方法的时候，需要用到这些字段，比如m_nodes以及m_independentNodes这两个集合，它们分别存放的是所有的节点的引用以及所有独立节点的引用。
- Graph对象是如何使用的，即方法调用栈的上层是如何调用Graph中的topologicalSort方法的。
- 关于环路检测算法的实现，用于在发现循环依赖的时候，检测出具体的循环依赖路径。

本文就对上述的几个问题作出解释：

<!-- more -->

## 环路检测算法

先来看看算法的实现：

```java
public class Tarjan<T> {  
  int m_index = 0;  
  private Stack<T> m_s;  
  Map<T, Integer> m_indices = Maps.newHashMap();  
  Map<T, Integer> m_lowlinks = Maps.newHashMap(); //  
  private List<T> m_cycle;    
  public Tarjan(Graph<T> graph, T start){  
    m_s = new Stack<T>();  
    run(graph, start);  
  }  
  private void run(Graph<T> graph, T v) {  
    m_indices.put(v, m_index);  
    m_lowlinks.put(v, m_index);  
    m_index++;  
    m_s.push(v);  
    for (T vprime : graph.getPredecessors(v)){  
      if (! m_indices.containsKey(v prime)) {  
        run(graph, vprime);  
        int min = Math.min(m_lowlinks.get(v),m_lowlinks.get(vprime));  
        m_lowlinks.put(v, min);  
      }  
      else if (m_s.contains(v prime)) {  
        m_lowlinks.put(v,Math.min(m_lowlinks.get(v), m_indices.get(vprime)));  
      }  
    }  
    if (m_lowlinks.get(v) == m_indices.get(v)){  
      m_cycle = Lists.newArrayList();  
      T n;  
      do {  
        n = m_s.pop();  
        m_cycle.add(n);  
      } while (! n.equals(v));  
    }  
  }  
  public List<T> getCycle() {  
    return m_cycle;  
  }  
}  
```

这里实现的实际上是Tarjan判断强连通子图的算法，因为对于有向cycle，它一定是强连通的，所以在我们的场景中使用这个算法是没问题的，但是在细节上，上面的实现存在一点小瑕疵，即最后只能maintain一个cycle，如果在依赖关系中存在多个cycle的话，是无法将它们全部记录下来的。当然，这个算法的实现是为了提示用户存在循环依赖，而不是为了输出所有的循环依赖。有兴趣的可以查看维基百科中对于[Tarjan SCC算法](http://en.wikipedia.org/wiki/Tarjan%27s_strongly_connected_components_algorithm)的描述。

另外，在上面的实现中，和Tarjan SCC算法的实现也不是完全一致的，比如，是对当前node的所有前驱结点进行检查，而不是像Tarjan SCC中，是对所有的可达节点进行检查。这样做也是为了加快算法的执行速度，即在存在循环依赖的情况下，尽量减少循环的次数，只需要保证至少能够检测到一个环即可。

## Graph对象的初始化以及使用

我们再来回顾一下那个stacktrace：

![](http://my.csdn.net/uploads/201206/07/1339047914_1787.png)

可以发现，调用Graph中的拓扑排序方法的是MethodHelper类中的静态方法topologicalSort，而后者又被该类中的几个静态方法调用，所以，为了弄清楚Graph中的数据是如何准备的，我们需要对这个类进行探究：

### MethodHelper.topologicalSort方法

```java
private static Graph<ITestNGMethod> topologicalSort(ITestNGMethod[] methods,  
      List<ITestNGMethod> sequentialList, List<ITestNGMethod> parallelList) {  
    Graph<ITestNGMethod> result = new Graph<ITestNGMethod>();  // 首先创建一个Graph类，Graph类中只有一个默认的constructor  
    if (methods.length == 0) {    
      return result;   //如果传入的methods数组长度为0，直接返回了空的graph  
    }  
    // Create the graph  
    for (ITestNGMethod m : methods) {  
      result.addNode(m);    //对于每个方法instance，添加到graph中  
      List<ITestNGMethod> predecessors = Lists.newArrayList();  //获得该方法依赖的方法，获得该方法依赖的group名字，返回的都是string数组  
      String[] methodsDependedUpon = m.getMethodsDependedUpon();  
      String[] groupsDependedUpon = m.getGroupsDependedUpon();  
      if (methodsDependedUpon.length > 0) {   //如果存在依赖的方法  
        ITestNGMethod[] methodsNamed =  
          MethodHelper.findDependedUponMethods(m,methods);  // 通过该静态方法找到相应的method instances  
        for (ITestNGMethod pred : methodsNamed){  
         predecessors.add(pred);   //将找到的依赖方法添加到pred list中  
        }  
      }  
      if (groupsDependedUpon.length > 0) {  
        for (String group : groupsDependedUpon){ // 对于每个组，找到组中的方法  
          ITestNGMethod[] methodsThatBelongToGroup =  
            MethodGroupsHelper.findMethodsThatBelongToGroup(m, methods, group);  
          for (ITestNGMethod pred : methodsThatBelongToGroup) {  
            predecessors.add(pred); // 将找到的依赖方法添加到pred list中  
          }  
        }  
      }  
      for (ITestNGMethod predecessor : predecessors) {  
        result.addPredecessor(m, predecessor);  // 将pred list中的方法instance添加到graph中  
      }  
    }  
    result.topologicalSort();    //调用Graph的TS方法  
    sequentialList.addAll(result.getStrictlySortedNodes());    //对于存在依赖关系的方法需要顺序运行  
    parallelList.addAll(result.getIndependentNodes());    //对于不存在任何依赖的方法可以并发运行  
    return result;  
}  
```

以上代码的几个关键步骤：

- 对于每个ITestNGMethod对象的操作：
	- 添加到Graph中，通过addNode方法
	- 创建一个list，用来维护该方法依赖的方法
	- 获得该方法依赖的所有方法，添加到上一步创建的list中
		- 根据dependsOnMethod找到依赖方法
		- 根据dependsOnGroup找到依赖方法
	- 将上一步修改后的list添加到graph中，通过addPredecessor方法
- 待所有的ITestNGMethod对象都被处理完毕后，调用Graph对象的拓扑排序方法
- 如果拓扑排序没有出现错误，获取结果，分别添加到Sequential和Parallel list中

在上面的分析中，出现了SequentialList以及Parallel List这两个集合，它们分别用于顺序执行和并发执行。并发执行是TestNG中一个很重要，同时也十分新颖的功能。我们总是希望最大限度的提高程序的并发度，对于测试用例的运行，也不例外。由于硬件的发展，并发/并行计算是未来的趋势之一。TestNG中对于并发运行功能的实现，以后会有介绍。

在上一篇文章中，介绍了Graph类的工作原理，但是对于其中数据的来源和准备，当时我们暂时忽略了，那么现在我们可以详细探究一下，Graph中的数据是如何准备的：

主要通过两个方法：

`addNode`以及`addPredecessor`

由于Graph是一个泛型类，这里的参数都用T来表示类型，为了方便理解，不妨把这个T就当成TestNG中的用来表示方法的`ITestNGMethod`接口类型。

```java
public void addNode(T tm) {  
  ppp("ADDING NODE " + tm + "" + tm.hashCode());  
  m_nodes.put(tm, new Node<T>(tm)); // Initially, all the nodes are put in theindependent list as well  
}  
```

该方法的实现十分简单，就是向m_nodes集合中添加一个entry，注意这个entry的类型是<Method，Node<Method>>:

```java
public void addPredecessor(T tm, T predecessor) {  
  Node<T> node = findNode(tm);  // 首先看看是否能够得到tm对象对应的Node  
  if (null == node) {  
    throw new TestNGException("Non-existing node: " + tm); //如果没有找到，明显是发生了错误，代表这个tm在m_nodes中根本就不存在  
  }  
  else {  
    node.addPredecessor(predecessor);  // 这里在node上调用了addPredecessor方法  
    addNeighbor(tm, predecessor);   //这个方法的作用暂时还没有看到，搜索了整个workspace都没有找到相应的用途，估计是作者为了扩展什么功能而预留的  
    // Remove these two nodes from theindependent list  
    if (null == m_independentNodes) {  // 如果独立节点集合还没有初始化  
      m_independentNodes = Maps.newHashMap();  
      for (T k : m_nodes.keySet()) { // 将现有的所有方法instance以及对应的Node对象添加到独立map  
        m_independentNodes.put(k,m_nodes.get(k));    
      }  
    }  // 移除非独立的节点，包括tm对象本身以及它依赖的节点，因此该集合中最后剩下的就是完全独立的节点了  
    m_independentNodes.remove(predecessor);             
    m_independentNodes.remove(tm);     
    ppp("  REMOVED " + predecessor + " FROMINDEPENDENT OBJECTS");  
  }  
}  
```

然后我们再看看上面方法的调用者：

### MethodHelper.sortMethods方法

```java
private static List<ITestNGMethod> sortMethods(boolean forTests,  
    List<ITestNGMethod>allMethods, IAnnotationFinder finder) {  
  List<ITestNGMethod>sl = Lists.newArrayList();  // 用来保存只能顺序执行的methods  
  List<ITestNGMethod>pl = Lists.newArrayList(); // 用来保存可以并行执行的methods  
  ITestNGMethod[] allMethodsArray =allMethods.toArray(new ITestNGMethod[allMethods.size()]);  
  
   // 下面这个if块的功能在于：对属于beforeXXX系列的configuration方法，在调用顺序上进行修正  
  if (!forTests && allMethodsArray.length > 0) {   
    ITestNGMethod m = allMethodsArray[0];  
    boolean before = m.isBeforeClassConfiguration()  
        || m.isBeforeMethodConfiguration() || m.isBeforeSuiteConfiguration()  
        || m.isBeforeTestConfiguration();    //查看该config方法是不是beforeXXX方法  
    MethodInheritance.fixMethodInheritance(allMethodsArray, before);  
  }  // 调用之前的topologicalSort方法进行拓扑排序，将sequential list和parallel list构造好  
  topologicalSort(allMethodsArray, sl, pl);   
  List<ITestNGMethod> result = Lists.newArrayList();  
  result.addAll(sl);  
  result.addAll(pl);  // 将sl和pl中的ref全部添加到result中，该result集合也就表示了所有需要执行的方法  
  return result;  
}  
```

因此，在TestNG对于依赖关系检测的拓扑排序中，主要有两个功能：

- 检测依赖关系的正确性，即不存在任何形式的循环依赖
- 在保证正确性的前提下，将方法分类，分成只能顺序运行的方法以及可以并发运行的方法

以上，就是对TestNG中依赖关系相关核心代码的分析。其核心思想还是使用拓扑排序来建立依赖关系。在以后的系列文章中，还会介绍TestNG是如何实现并发运行测试方法，以及一些其他内容，比如，TestNG的几个常用的扩展点，Method Selector机制，各种Listener等等。


