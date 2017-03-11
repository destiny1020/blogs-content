---
title: "TestNG 并发运行相关的核心概念 - 补充"
date: 2012-06-15 23:07:00
categories: [译文, TestNG]
tags: [TestNG, 并发]
---

翻译自：http://beust.com/weblog/2009/12/13/more-on-multithreaded-topological-sorting-2/ 

**限于翻译水平有限，有兴趣的同学可以直接看原文。翻译上如果有什么不妥之处，还麻烦指正，感谢！**

![](http://my.csdn.net/uploads/201206/15/1339772470_1861.png)

在上篇文章中我介绍了在TestNG中新实现的多线程拓扑排序(译注：严格说来，应该是更加动态的拓扑排序，为了支持对待测试方法的并发运行，上一篇文章的译文连接在[这里](http://www.rxjiang.com/2012/06/14/TestNG-%E5%B9%B6%E5%8F%91%E8%BF%90%E8%A1%8C%E7%9B%B8%E5%85%B3%E7%9A%84%E6%A0%B8%E5%BF%83%E6%A6%82%E5%BF%B5/))，这篇文章收到了一些有意思的评论，其中有一条来自Rafael Naufal的评论我想拿出来讨论一下：

<!-- more -->

他提到：

@Priority注解难道不能改进，从而能够表明哪个独立方法应该被首先调度？顺便说一下，判断哪些方法是独立的难道不能放到测试方法的graph对象中么？

这个想法让当前的算法更进一步，即我们不仅需要在测试方法变成独立节点的时候去调度它们，最好还能够根据它们重要的程度去调度。

什么让一个节点比其它的节点更加重要？是它的被依赖程度。一个节点被依赖的越多，那么尽快调度这个节点就更有益，因为这个节点一旦运行结束，就能够让更多的节点变成独立状态。诚然，你还是会受到线程池大小的限制(***译注：即最大并发数仍受到线程池大小的限制***)，但是这就是我们需要的：增加线程池的大小就能够增加并发度，因此也就带来了更好的性能，但是现在公平的(或者称为随机的)调度算法并不能保证性能会增加。

我的第一个反应就是修改我的Executor，但是实际上你能够通过现有的实现来达到目的。[Executors](http://docs.oracle.com/javase/1.5.0/docs/api/java/util/concurrent/ThreadPoolExecutor.html)的构造方法中接受一个BlockingQueue作为参数，Executor会根据这个queue来处理其中的workers。毫不意外地，现在已经有一个优先队列的实现，叫做[PriorityBlockingQueue](http://docs.oracle.com/javase/1.5.0/docs/api/java/util/concurrent/PriorityBlockingQueue.html)。

你需要做的就是在创建你自己的Executor时，使用这个带有queue作为参数的构造方法，而不是默认的构造方法，然后还需要保证你传入的workers需要拥有自然排序(***译注：其实也不一定需要自然排序，显式的指定一个Comparator也是可行的，这是因为PriorityBlockingQueue需要知道如何确定优先级，而此情此景的优先级，就是worker的权重，如后文***)。在这种情况下，worker的权重就是有多少其他的workers依赖它，这是非常容易计算的。

与此相关的，我想仔细看看在我上一篇博文中提到的算法是如何工作的。我描述了一些理论，同时我对该算法进行了一些测试，算法也符合预期，但是我突然想起来我能够很轻松的从细节的角度去考虑这些问题。(***译注：因为这些代码本来就是博主写的……，他的意思应该是他没有考虑到读者可能对细节不太熟悉***)

首先，我添加了一个toDot方法，用来产生一个表示当前图的[Graphviz](http://www.graphviz.org/)文件，这很容易实现：

```java
/** 
* @return a .dot file (GraphViz) version of this graph. 
*/  
public String toDot() {  
  String FREE = "[style=filled color=yellow]";  
  String RUNNING = "[style=filled color=green]";  
  String FINISHED = "[style=filled color=grey]";  
  StringBuilder result = new StringBuilder("digraph g {\n");  
  Set<T> freeNodes = getFreeNodes();  
  String color;  
  for (T n : m_nodesReady) {  
    color = freeNodes.contains(n) ? FREE : "";  
    result.append("  " + getName(n) + color + "\n");  
  }  
  for (T n : m_nodesRunning) {  
    color = freeNodes.contains(n) ? FREE : RUNNING;  
    result.append("  " + getName(n) + color + "\n");  
  }  
  for (T n : m_nodesFinished) {  
    result.append("  " + getName(n) + FINISHED+ "\n");  
  }  
  result.append("\n");  
  for (T k : m_dependingOn.getKeys()) {  
    List<T> nodes = m_dependingOn.get(k);  
    for (T n : nodes) {  
      String dotted = m_nodesFinished.contains(k) ? "style=dotted" : "";  
      result.append("  " + getName(k) + " -> " + getName(n) + " [dir=back " + dotted +"]\n");  
    }  
  }  
  result.append("}\n");  
  return result.toString();  
}  
```

然后我修改executor，让它在每个worker终止的时候就绘制一张图，最后，我写了一个shell脚本用来将这些dot文件转换为图像并生成了一个HTML文件。我跑了一个简单的测试，通过该shell脚本处理，最后的得到的结果在[这里](http://beust.com/topo/graphs.html)。

黄色的节点是独立的，绿色代表该节点处于就绪状态(可以在线程池中运行了)，灰色代表完成状态，白色则指该节点还没有被处理。点划线的箭头表示已经满足的依赖关系。

正如你看到的，执行的结果与我们描述的算法的工作模式十分接近。同时我也确认了改变线程池的大小能够产生不同的执行方式。
