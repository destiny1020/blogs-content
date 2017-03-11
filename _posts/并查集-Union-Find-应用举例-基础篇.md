---
title: 并查集(Union-Find) 应用举例 --- 基础篇
date: 2012-07-22 23:54:00
categories: [算法, 图论]
tags: [算法, 图论, 并查集]
---

本文是作为上一篇文章 [并查集算法原理和改进](http://www.rxjiang.com/2012/06/12/%E5%B9%B6%E6%9F%A5%E9%9B%86-Union-Find-%E7%AE%97%E6%B3%95%E4%BB%8B%E7%BB%8D/) 的后续，焦点主要集中在一些并查集的应用上。材料主要是取自POJ，HDOJ上的一些算法练习题。

<!-- more -->

1. 以树作为节点的组织结构，结构的形态很是否采取优化策略有很大关系，未进行优化的树结构可能会是“畸形”树(严重不平衡，头重脚轻，退化成链表等)，按尺寸(正规说法叫做秩，后文全部用秩来表示)进行平衡，同时辅以路径压缩后，树结构会高度扁平化。
2. 虽然组织结构比较复杂，数据表示方式却十分简洁，主要采用数组作为其底层数据结构。一般会使用两个数组(parent-link array and size array)，分别用来保存当前节点的父亲节点以及当前节点所代表子树的秩。第一个数组(parent-link array)无论是否优化，都需要使用，而第二个数组(size array)，在不需要按秩合并优化或者不需要保存子树的秩时，可以不使用。根据应用的不同，可能需要第三个数组来保存其它相关信息，比如HDU-3635中提到的“转移次数”。
3. 主要操作包括两部分，union以及find。union负责对两颗树进行合并，合并的过程中可以根据具体应用的性质选择是否按秩优化。需要注意的是，执行合并操作之前，需要检查待合并的两个节点是否已经存在于同一颗树中，如果两个节点已经在一棵树中了，就没有合并的必要了。这是通过比较两个节点所在树的根节点来实现的，而寻找根节点的功能，自然是由find来完成了。find通过parent-link数组中的信息来找到指定节点的根节点，同样地，也可以根据应用的具体特征，选择是否采用路径压缩这一优化手段。然而在需要保存**每个**节点代表子树的秩的时候，则无法采用路径压缩，因为这样会破坏掉非根节点的尺寸信息(**注意这里的“每个”，一般而言，在按秩合并的时候，需要的信息仅仅是根节点的秩，这时，路径压缩并无影响，路径压缩影响的只是非根节点的秩信息**)。

以上就是我认为并查集中存在的几个关键点。关于并查集更详尽的演化过程，可以参考上一篇关于并查集的文章：[并查集算法原理和改进](http://www.rxjiang.com/2012/06/12/%E5%B9%B6%E6%9F%A5%E9%9B%86-Union-Find-%E7%AE%97%E6%B3%95%E4%BB%8B%E7%BB%8D/)

言归正传，来看几个利用并查集来解决问题的例子：
(说明：除了第一个问题贴了完整的代码，后面的问题都只会贴出关键部分的代码)

## HDU-1213 How many tables

问题的描述是这样的：

Today is Ignatius' birthday. He invites a lot of friends. Now it's dinner time. Ignatius wants to know how many tables he needs at least. You have to notice that not all the friends know each other, and all the friends do not want to stay with strangers.

One important rule for this problem is that if I tell you A knows B, and B knows C, that means A, B, C know each other, so they can stay in one table.

For example: If I tell you A knows B, B knows C, and D knows E, so A, B, C can stay in one table, and D, E have to stay in the other one. So Ignatius needs 2 tables at least.

对这个问题抽象之后，就是要求进行若干次union操作之后，还会剩下多少颗树(**或者说还剩下多少Connected Components**)。反映到这个例子中，就是要求有多少“圈子”。其实，这也是社交网络中的最基本的功能，每次系统向你推荐的那些好友一般而言，会跟你在一个“圈子”里面，换言之，也就是你可能认识的人，以并查集的视角来看这层关系，就是你们挂在同一颗树上。

给出实现代码如下：

```java
import java.io.BufferedReader;  
import java.io.IOException;  
import java.io.InputStreamReader;  
import java.io.PrintWriter;  
  
public class Main {  
  
    public static void main(String[] args) throws NumberFormatException,  
            IOException {  
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));  
        PrintWriter out = new PrintWriter(System.out);  
  
        int totalCases = Integer.parseInt(br.readLine());  
  
        WeightedQUWithPathCompression uf;  
  
        String[] parts;  
        while (totalCases > 0) {  
            parts = br.readLine().split(" ");  
            // based on 1, not 0  
            uf = new WeightedQUWithPathCompression(  
                    Integer.parseInt(parts[0]) + 1);  
            // construct the uf  
            int tuples = Integer.parseInt(parts[1]);  
            while (tuples > 0) {  
                parts = br.readLine().split(" ");  
                uf.union(Integer.parseInt(parts[0]), Integer.parseInt(parts[1]));  
                tuples--;  
            }  
            out.println(uf.count() - 1);  
            br.readLine();  
            totalCases--;  
        }  
        out.flush();  
    }  
}  
  
class WeightedQUWithPathCompression {     
    private int count;  
    private int[] id;  
    private int[] size;  
  
    public WeightedQUWithPathCompression(int N) {  
        this.count = N;  
        this.id = new int[N];  
        this.size = new int[N];  
  
        for (int i = 0; i < this.count; i++) {  
            id[i] = i;  
            size[i] = 1;  
        }  
    }  
  
    private int find(int p) {  
        while (p != id[p]) {  
            id[p] = id[id[p]];  // 路径压缩，会破坏掉当前节点的父节点的尺寸信息，因为压缩后，当前节点的父节点已经变了  
            p = id[p];  
        }  
  
        return p;  
    }  
  
    public void union(int p, int q) {  
        int pCom = this.find(p);  
        int qCom = this.find(q);  
  
        if (pCom == qCom) {  
            return;  
        }  
        // 按秩进行合并  
        if (size[pCom] > size[qCom]) {  
            id[qCom] = pCom;  
            size[pCom] += size[qCom];  
        } else {  
            id[pCom] = qCom;  
            size[qCom] += size[pCom];  
        }  
        // 每次合并之后，树的数量减1  
        count--;  
    }  
  
    public int count() {  
        return this.count;  
    }  
}  
```

最后，通过调用count方法获取的返回值就是树的数量，也就是“圈子”的数量。

根据问题的具体特性，上面同时采用了两种优化策略，即按秩合并以及路径压缩。因为问题本身对合并的先后关系以及子树的秩这类信息不敏感。然而，并不是所有的问题都这样，比如下面这一道题目，他对合并的先后顺序就有要求。

## HDU-3635 Dragon Balls

该题的链接：[HDU-3635](http://acm.hdu.edu.cn/showproblem.php?pid=3635)

题意：起初球i是被放在i号城市的，在年代更迭，世事变迁的情况下，球被转移了，而且转移的时候，连带该城市的所有球都被移动了：T A B（A球所在的城市的所有球都被移动到了B球所在的城市），Q A（问：A球在那城市？A球所在城市有多少个球呢？A球被转移了多少次呢？）(上面题意的描述摘自[这里](http://www.cnblogs.com/Shirlies/archive/2012/03/06/2382118.html))

在这道题中，对子树进行合并时，就不能按秩进行合并，因为合并是有先后关系的。

我们重点关注一下要回答的问题是什么，比如Q A代表的问题就是：

A球在哪里？ --- 这个问题好回答，A球所在的城市就是该子树的根节点，即find方法的返回值。

A球所在的城市有多少个球？ --- 同样地，这个问题的答案就是size数组中对应位置的信息，虽然本题不能按秩进行合并优化，但是秩还是需要被保存下来的。

A球被转移了多少次呢？ --- 这个问题画张图，就比较好理解了( 请忽略绘图水平:) )：

![](http://my.csdn.net/uploads/201207/22/1342965314_9760.png)

首先将球1所在城市的所有球转移到球2所在的城市中，即城市2，然后将球1所在城市的所有球转移到球3所在的城市中，即城市3。显然，在第二步中，1球已经不在城市1中，因为其在第一步中已经转移到城市2了。然后第二步实际就是将城市2中的所有球(包括球1和球2)都转移到城市3中。

![](http://my.csdn.net/uploads/201207/22/1342965708_8741.png)

紧接着，将1球所在城市的球全部转移(包括球1,2,3)到球4所在的城市中，即是将3和4进行合并。这个时候如果直接进行合并的话，会得到一个链表状的结构，这种结构使我们一直都力求避免的，所以可以采用前面使用的路径压缩进行优化。路径压缩的具体做法就不赘述了。现在需要考虑的是，经过这3轮合并，球1到底移动了多少次？如果从最后的结果图来看，球1最后到城市4，应该移动了2次，即1->3, 3->4。但是，仔细想想就会发现，这是不正确的。因为在T1 2中球1首先移动到了城市2，然后T 1 3，表示1球所在的城市中的所有球被移动到了城市3中，即城市2中的球移动到城市3中，这会对1球进行一次移动。以此类推，最后在T 1 4中，1球从城市3中移动到了城市4中，又发生了一次移动，因此，1球一共移动了3次，1->2, 2->3, 3->4。那么这就存在问题了，至少在最后的图中，这一点很不直观，因为从1到4的路径上，已经没有2的踪迹了。显然，这是路径压缩带来的副作用。因为采用了路径压缩，所以对树结构造成了一些破坏，具体而言，是能够推导出球的转移次数的信息被破坏了。试想一下，如果没有进行路径压缩，转移次数实际上是很直观的，从待求节点到根节点走过的路径数，就是转移次数。

所以为了解决引入路径压缩带来的问题，需要引入第三个数组来保存每个球的转移次数。结合题意，每次在进行转移的时候，是转移该球所在城市中所有的球到目标球所在的城市，把这句话抽象一下，就是只有根节点才能够进行合并。因此，现有的union方法还是适用的，因为它在进行真正的合并之前，还是需要首先找到两个待合并节点的根节点。然后合并的时候，将第一个球所在城市的的号码的转移次数加1。按照这种想法，实现代码为：

```java
private static void union(int p, int q) {  
  int pRoot = find(p);  
  int qRoot = find(q);  
  
  if (pRoot == qRoot) {  
     return;  
  }  
  
  // 不能进行按秩合并，且在合并时，对第一个球的转移次数进行递增  
  id[pRoot] = qRoot;  
  trans[pRoot]++;  
  size[qRoot] += size[pRoot];  
}  
```

但是跟踪一下以上代码的调用过程不难发现，最后的球1，2，3，4的转移次数分别为1，1，1，0(唯一对trans数组进行影响的操作目前只存在于union方法中，见上)。显然，这是不正确的，正确的转移次数应该是3，2，1，0。那么是什么地方出了岔子呢，还是看看路径压缩就明白了，在路径压缩的时候，只顾着压缩，而没有对转移次数进行更新。

那么如何进行更新呢？看看上图，1本来是2的孩子，现在却成了3的孩子，跳过了2，因此可以看成，1->2->3的路径被压缩成了1->3，即2->3的这条路径被压缩了。被压缩在了1->3中，因此更新的操作也就有了基本的想法，我们可以讲被压缩的那条路径中的信息增加到压缩后的结果路径中，对应前面的例子，我们需要把2->3的信息给添加到1->3，用代码来表示的话，就是：`trans[1] += trans[2];`

一般化后，实现代码如下所示：

```java
private static int find(int q) {  
  while (id[q] != id[id[q]]) {   //如果q不是其所在子树的根节点的直接孩子  
     trans[q] += trans[id[q]];   //更新trans数组，将q的父节点的转移数添加到q的转移数中  
     id[q] = id[id[q]];          //对其父节点到其爷爷节点之间的路径进行压缩  
  }  
  return id[q];  
}  
```

最后，如果需要获得球A的转移次数，直接获取trans[A]就OK了。

## HDU-1856 More is better

这道题目的目的是想知道经过一系列的合并操作之后，查询在所有的子树中，秩的最大值是多少，简而言之，就是最大的那颗子树包含了多少个节点。

很显然，这个问题也能够同时使用两种优化策略，只不过因为要求最大秩的值，需要有一个变量来记录。那么在哪个地方来更新它是最好的呢？我们知道，在按秩进行合并的时候，需要比较两颗待合并子树的秩，因此可以顺带的将对秩的最大值的更新也放在这里进行，实现代码如下：

```java
private static void union(int p, int q) {  
    int pRoot = find(p);  
    int qRoot = find(q);  
  
    if (pRoot == qRoot) {  
        return;  
    }  
  
    if (sz[pRoot] > sz[qRoot]) {  
        id[qRoot] = pRoot;  
        sz[pRoot] += sz[qRoot];  
        if (sz[pRoot] > max) {    // 如果合并后的树的秩比当前最大秩还要大，替换之  
            max = sz[pRoot];  
        }  
    } else {  
        id[pRoot] = qRoot;  
        sz[qRoot] += sz[pRoot];  
        if (sz[qRoot] > max) {    // 如果合并后的树的秩比当前最大秩还要大，替换之  
            max = sz[qRoot];  
        }  
    }  
}  
```

这样，在完成了所有的合并操作之后，max中保存的即为所需要的信息。

## HDU-1272 | HDU-1325 小希的迷宫 | Is it a tree ?

这两道题目的链接：[HDU-1272](http://acm.hdu.edu.cn/showproblem.php?pid=1272), [HDU-1325](http://acm.hdu.edu.cn/showproblem.php?pid=1325)

这两个问题都是判断是否合并后的结构是一棵树，即结构中应该没有环路，除此之外，还有边数和顶点数量的之间的关系，应该满足 `edges + 1 = nodes`。

对于并查集，后者可以通过检查最后的connected components的数量是否为1来确定。

当然，两者在题目描述上还是有一定的区别，前者是无向图，后者是有向图。但是对于使用并查集来实现时，这一点的区别仅仅体现在合并过程无法按秩优化了。其实，如果能够采用路径压缩，按秩优化的效果就不那么明显了，因为每次进行查询操作的时候，会对被查询的节点进行路径压缩(参见find方法)，可以说这是一种“懒优化”，或者叫做“按需优化”。而按秩合并则是一个主动优化的过程，每次进行合并的时候都会进行。而采用按秩合并优化，需要额外一个保存size信息的数组，在一些应用场景中，对size信息并不在意，因此为了实现可选的优化方法而增加空间复杂度，就有一些得不偿失了。并且，对于按秩合并以及路径压缩到底能够提高多少效率，我们目前也并不清楚，这里做个记号，以后有空了写一篇相关的文章。

扯远了，回到正题。前面提到了判断一张图是否是一颗树的两个关键点：

1. 不存在环路(对于有向图，不存在环路也就意味着不存在强连通子图)
2. 满足边数加一等于顶点数的规律(不考虑重边和指向自身的边)

第一条，在并查集中应该如何实现呢？

现在我们对并查集也有一定的认识了，其实很容易我们就能够想出，当两个顶点的根节点相同时，就代表添加了这一条边后会出现环路。这很好解释，如果两个顶点的根节点是相同的，代表这两个顶点已经是连通的了，对于已经连通的两个顶点，再添加一条边，必然会产生环路。

第二条呢？

图中的边数，我们可以在每次进行真正合并操作之前(也就是，在确认两个待合并的顶点的根节点不相同时)进行记录。然后顶点数，也就是整个合并过程中参与进来的顶点个数了，可以使用一个布尔数组来进行记录，出现后将相应位置设为true，最后进行一轮统计即可。

相关实现：

```java
private static void union(int p, int q) {  
   int pRoot = find(p);  
   int qRoot = find(q);  
  
   if (pRoot == qRoot) {  
      valid = false;  // 此处的valid是一个boolean变量，置为false表示改图不是一颗树  
      return;  
   }  
   mark[p] = true;  
   mark[q] = true;   // p和q参与到最后的顶点数量的统计  
   edges++;   // 在合并之前，将边的数量递增  
   id[qRoot] = pRoot;  
} 
```

## 总结

就目前看来，一般问题都是围绕着并查集的两个主要操作，union和find做文章，根据具体应用，增加一些信息，增加一些逻辑，例如上题中的转移次数，或者是根据问题特征选择使用合适的优化策略，按秩合并以及路径压缩。

并查集是我比较喜欢的一种数据结构和算法，很多实际问题都能够利用它给出高效而简洁的解决方案。后续如果有空还会陆续介绍一些有代表性的，同时难度也更大的题目，敬请关注。






