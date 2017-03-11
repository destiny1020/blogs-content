---
title: '[Elasticsearch] 分布式搜索'
date: 2014-11-19 10:10:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 分布式, 搜索]
---

## 分布式搜索

本文翻译自Elasticsearch官方指南的[Distributed Search Execution](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/distributed-search.html)一章。

在继续之前，我们将绕一段路来谈谈在分布式环境中，搜索是如何执行的。和在分布式文档存储(Distributed Document Store)中讨论的基本CRUD操作相比，这个过程会更加复杂一些。

一个CRUD操作会处理一个文档，该文档有唯一的_index，_type和路由值(Routing Value，它默认情况下就是文档的_id)组合。这意味着我们能够知道该文档被保存在集群中的哪个分片(Shard)上。

<!-- More -->

然而，搜索的执行模型会复杂的多，因为我们无法知道哪些文档会匹配 - 匹配的文档可以在集群中的任何分片上。因此搜索请求需要访问索引中的每个分片来得知它们是否有匹配的文档。

但是，找到所有匹配的文档只完成了一半的工作。从多个分片中得到的结果需要被合并和排序以得到一个完整的结果，然后search API才能够将结果返回。正因为如此，搜索由两个阶段组成(Two-phase Process) - 查询和获取(Query then Fetch)。

### 查询阶段(Query Phase)

在查询的初始阶段，查询会被广播(Broadcast)给索引中的每个分片拷贝(Shard Copy，它可以是主分片或者是副本分片)。然后每个分片会在本地执行该搜索，匹配的文档会被保存到一个优先队列(Priority Queue)中。

> 优先队列
> 
> 优先队列实际上是一个用来保存前N个匹配(Top-N Matching)的文档的有序列表。优先队列的大小取决于分页参数：from和size。比如，下面的搜索请求会建立一个大小为100的优先队列用于保存匹配的文档：
> 
> ```
> GET /_search
> {
>     "from": 90,
>     "size": 10
> }
> ```

查询阶段的过程如下图所示：

![](http://img.blog.csdn.net/20161202133354946?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

1. 客户端发送一个搜索请求到节点3，节点3随即会创建一个大小为from + size的空的优先队列。
2. 节点3将搜索请求转发到索引中每个分片的主分片(Primary Shard)或副本分片(Replica Shard)。然后每个分片会在本地执行该查询，然后将结果保存到本地的一个大小同样为from + size的优先队列中。
3. 每个分片返回优先队列中文档的IDs和它们的排序值到协调节点(Coordinating Node)，也就是节点3。然后节点3负责将所有的结果合并到本地的优先队列中，这个优先队列就是全局的查询结果。

当一个搜索请求被发往一个节点时，该节点就变成了协调节点(Coordinating Node)。它需要向其他关联分片所在的节点广播搜索请求，然后合并来自它们的中间结果来得到最终能够发送给客户端的全局结果。

第一步是将请求广播给索引中所有分片拷贝(可以是主分片，也可以是副本分片)所在的节点。和获取文档的GET请求一样，搜索请求也可以被主分片或者它关联的任意一个副本分片处理。这就是当增加副本分片(添加更多的硬件)能够增加搜索吞吐量的原由。协调节点会通过循环(Round-robin)的方式向所有的分片拷贝发送请求来分布负载。

每个分片会在本地执行查询，然后建立一个大小为from + size的优先队列来保存其结果。换言之，在本地就能够满足全局的搜索请求(译注：因为本地的优先队列大小和协调节点上的优先队列大小是一致的)。它会返回一个轻量级的结果列表给协调节点 - 仅包含文档的IDs和在排序过程中需要的相关值，比如_score。

协调节点会将从其他节点返回的结果合并来得到一个全局的优先队列。到这里，查询阶段就结束了。

> 多索引搜索(Multi-index Search)
> 
> 一个索引能够含有一个或者多个主分片，因此针对一个索引的搜索请求需要对来自多个分片的搜索结果进行合并。一个针对多个或者所有索引的搜索请求也以相同的方式工作 - 只是更多的分片会参与到这个过程中。

### 获取阶段(Fetch Phase)

查询阶段能够辨识出哪些文档能够匹配搜索请求，但是我们还需要获取文档本身。这就是获取阶段完成的工作。此阶段如下图所示：

![](http://img.blog.csdn.net/20161202133404758?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

1. 协调节点(Coordinating Node)会首先辨识出哪些文档需要被获取，然后发出一个multi GET请求到相关的分片。
2. 每个分片会读取相关文档并根据要求对它们进行润色(Enrich)，最后将它们返回给协调节点。
3. 一旦所有的文档都被获取了，协调节点会将结果返回给客户端。

协调节点首先决定到底哪些文档是真的需要被获取的。比如，如果在查询中指定了{ "from": 90, "size": 10 }，那么头90条结果都会被丢弃(Discarded)，只有剩下的10条结果所代表的文档需要被获取。这些文档可能来自一个或者多个分片。

协调节点会为每个含有目标文档的分片构造一个[multi GET](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/distrib-multi-doc.html)请求，然后将该请求发送给在查询阶段中参与过的分片拷贝。

分片会加载文档的正文部分 - 即_source字段 - 如果被要求的话，还会对结果进行润色，比如元数据和[搜索片段高亮(Search Snippet Highlighting)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/highlighting-intro.html)。一旦协调节点获取到了所有的结果，它就会将它们组装成一个响应并返回给客户端。

> 深度分页(Deep Pagination)
> 
> 查询并获取(Query then Fetch)的过程支持通过传入的from和size参数来完成分页功能。但是该功能存在限制。不要忘了每个分片都会在本地保存一个大小为from + size的优先队列，该优先队列中的所有内容都需要被返回给协调节点。然后协调节点需要对number_of_shards * (from + size)份文档进行排序来保证最后能够得到正确的size份文档。
> 
> 根据文档的大小，分片的数量以及你正在使用的硬件，对10000到50000个结果进行分页(1000到5000页)是可以很好地完成的。但是当from值大的一定程度，排序就会变成非常消耗CPU，内存和带宽等资源的行为。因为如此，我们强烈地建议你不要使用深度分页。
> 
> 实践中，深度分页返回的页数实际上是不切实际的。用户往往会在浏览了2到3页后就会修改他们的搜索条件。罪魁祸首往往都是网络爬虫这种无休止地页面进行抓取的程序，它们会让你的服务器不堪重负。
> 
> 如果你确实需要从集群中获取大量的文档，那么你可以使用禁用了排序功能的scan搜索类型来高效地完成该任务。我们会在稍后的Scan和Scroll一节中进行讨论。

### 搜索选项(Search Options)

有一些搜索字符串参数能够影响搜索过程：

#### preference

该参数能够让你控制哪些分片或者节点会用来处理搜索请求。它能够接受：_primary，_primary_first，_local，_only_node:xyz，_prefer_node:xyz以及_shards:2,3这样的值。这些值的含义在[搜索preference](http://www.elasticsearch.org/guide/en/elasticsearch/reference/1.4//search-request-preference.html)的文档中有详细解释。

但是，最常用的值是某些任意的字符串，用来避免结果跳跃问题(Bouncing Result Problem)。

> 结果跳跃(Bouncing Results)
> 
> 比如当你使用一个timestamp字段对结果进行排序，有两份文档拥有相同的timestamp。因为搜索请求是以一种循环(Round-robin)的方式被可用的分片拷贝进行处理的，因此这两份文档的返回顺序可能因为处理的分片不一样而不同，比如主分片处理的顺序和副本分片处理的顺序就可能不一样。
> 
> 这就是结果跳跃问题：每次用户刷新页面都会发现结果的顺序不一样。
> 
> 这个问题可以通过总是为相同用户指定同样的分片来避免：将preference参数设置为一个任意的字符串，比如用户的会话ID(Session ID)。

#### timeout

默认情况下，协调节点会等待所有分片的响应。如果一个节点有麻烦了，那么会让所有搜索请求的响应速度变慢。

timeout参数告诉协调节点它在放弃前要等待多长时间。如果放弃了，它会直接返回当前已经有的结果。返回一部分结果至少比什么都不返回要强。

在搜索请求的响应中，有用来表明搜索是否发生了超时的字段，同时也有多少分片成功响应的字段：

```json
...
    "timed_out":     true,  
    "_shards": {
       "total":      5,
       "successful": 4,
       "failed":     1 
    },
...
```

#### routing

在分布式文档存储(Distributed Document Store)一章中的路由文档到分片(Routing a document to a shard)一小节中，我们已经解释了自定义的routing参数可以在索引时被提供，来确保所有相关的文档，比如属于同一用户的文档，都会被保存在一个分片上。在搜索时，相比搜索索引中的所有分片，你可以指定一个或者多个routing值来限制搜索的范围到特定的分片上：

```
GET /_search?routing=user_1,user2
```

该技术在设计非常大型的搜索系统有用处，在[Designing for scale](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/scale.html)中我们会详细介绍。

#### search_type

除了query_then_fetch是默认的搜索类型外，还有其他的搜索类型可以满足特定的目的，比如：

```
GET /_search?search_type=count
```

**count**

count搜索类型只有查询阶段。当你不需要搜索结果的时候可以使用它，它只会返回匹配的文档数量或者聚合(Aggregation)结果。

**query_and_fetch**

query_and_fetch搜索类型会将查询阶段和获取阶段合并成一个阶段。当搜索请求的目标只有一个分片时被使用，比如指定了routing值时，是一种内部的优化措施。尽管你可以选择使用该搜索类型，但是这样做几乎是没有用处的。

**dfs_query_then_fetch和dfs_query_and_fetch**

dfs搜索类型有一个查询前阶段(Pre-query phase)用来从相关分片中获取到词条频度(Term Frequencies)来计算群安居的词条频度。我们会在[相关性错了(Relevance is broken)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/relevance-is-broken.html)一节中进行讨论。

**scan**

scan搜索类型会和scroll API一起使用来高效的获取大量的结果。它通过禁用排序来完成。在下一小节中会进行讨论。

### scan和scroll

scan搜索类型以及scroll API会被一同使用来从ES中高效地获取大量的文档，而不会有深度分页(Deep Pagination)中存在的问题。

#### scroll

一个滚动(Scroll)搜索能够让我们指定一个初始搜索(Initial Search)，然后继续从ES中获取批量的结果，直到所有的结果都被获取。这有点像传统数据库中的游标(Cursor)。

一个滚动搜索会生成一个实时快照(Snapshot) - 它不会发现在初始搜索后，索引发生的任何变化。它通过将老的数据文件保存起来来完成这一点，因此它能够保存一个在它开始时索引的视图(View)。

#### scan

在深度分页中最耗费资源的部分是对全局结果进行排序，但是如果我们禁用了排序功能的话，就能够快速地返回所有文档了。我们可以使用scan搜索类型来完成。它告诉ES不要执行排序，只是让每个还有结果可以返回的分片返回下一批结果。

---

为了使用scan和scroll，我们将搜索类型设置为scan，同时传入一个scroll参数来告诉ES，scroll会开放多长时间：

```json
GET /old_index/_search?search_type=scan&scroll=1m 
{
    "query": { "match_all": {}},
    "size":  1000
}
```

以上请求会让scroll开放一分钟。

此请求的响应不会含有任何的结果，但是它会含有一个_scroll_id，它是一个通过Base64编码的字符串。现在可以通过将_scroll_id发送到_search/scroll来获取第一批结果：

```
GET /_search/scroll?scroll=1m 
c2Nhbjs1OzExODpRNV9aY1VyUVM4U0NMd2pjWlJ3YWlBOzExOTpRNV9aY1VyUVM4U0 
NMd2pjWlJ3YWlBOzExNjpRNV9aY1VyUVM4U0NMd2pjWlJ3YWlBOzExNzpRNV9aY1Vy
UVM4U0NMd2pjWlJ3YWlBOzEyMDpRNV9aY1VyUVM4U0NMd2pjWlJ3YWlBOzE7dG90YW
xfaGl0czoxOw==
```

该请求会让scroll继续开放1分钟。_scroll_id能够通过请求的正文部分，URL或者查询参数传入。

注意我们又一次指定了?scroll=1m。scroll的过期时间在每次执行scroll请求后都会被刷新，因此它只需要给我们足够的时间来处理当前这一批结果，而不是匹配的所有文档。

这个scroll请求的响应包含了第一批结果。尽管我们指定了size为1000，我们实际上能够获取更多的文档。size会被每个分片使用，因此每批最多能够获取到size * number_of_primary_shards份文档。

> NOTE
> 
> scroll请求还会返回一个新的_scroll_id。每次我们执行下一个scroll请求时，都需要传入上一个scroll请求返回的_scroll_id。

当没有结果被返回是，我们就处理完了所有匹配的文档。

> TIP
> 
> 一些ES官方提供的客户端提供了scan和scroll的工具用来封装这个功能。
