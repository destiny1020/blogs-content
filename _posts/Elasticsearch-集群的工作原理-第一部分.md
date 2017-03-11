---
title: '[Elasticsearch] 集群的工作原理 - 第一部分'
date: 2014-11-17 10:25:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 集群]
---

本文翻译自Elasticsearch官方指南的[life inside a cluster](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/distributed-cluster.html)一章。

ES就是为高可用和可扩展而生的。扩展可以通过购置性能更强的服务器(垂直扩展或者向上扩展，Vertical Scale/Scaling Up)，亦或是通过购置更多的服务器(水平扩展或者向外扩展，Horizontal Scale/Scaling Out)来完成。

尽管ES能够利用更强劲的硬件，垂直扩展毕竟还是有它的极限。真正的可扩展性来自于水平扩展 - 通过向集群中添加更多的节点来分布负载，增加可靠性。

在大多数数据库中，水平扩展通常都需要你对应用进行一次大的重构来利用更多的节点。相反，ES天生就是分布式的：它知道如何管理多个节点来完成扩展和实现高可用性。这也意味着你的应用不需要在乎这一点。

在本章中，我们会介绍如何建立集群(Cluster)，节点(Node)和分片(Shard)来根据你的需求完成扩展，同时也能够保证即使发生硬件故障你的数据也会安然无恙。

<!-- More -->

## 一个空的集群

如果我们启动了一个没有任何数据和索引的节点(Node)，我们的集群(Cluster)看起来就像下面这样：

![](http://img.blog.csdn.net/20161202131927394?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

一个节点会运行一个ES的实例，而一个集群则会包含拥有相同cluster.name的一个或者多个节点，这些节点共同工作来完成数据共享和负载分担。随着节点被添加到集群，或者从集群中被删除，集群会通过自身调节来将数据均匀分布。

集群中的一个节点会被选为主节点(Master Node)，它负责管理整个集群的变化，如创建或者删除一个索引(Index)，向集群中添加或者删除节点。主节点并不需要参与到文档级别的变化或者搜索中，这意味着虽然只有一个主节点，但它并不会随着流量的增加而成为瓶颈。任何节点都可以成为主节点。在我们的例子中只有一个节点，所以它就承担了主节点的功能。

对于用户，可以和集群中的任意节点进行通信，包括主节点。每个节点都知道每份文档的存放位置，并且能够将请求转发到持有所需数据的节点。用户通信的节点会负责将需要的数据从各个节点收集起来，然后返回给用户。以上整个过程都会由ES透明地进行管理。

## 集群健康指标(Cluster Health)

在一个ES集群中，有很多可以被监测的统计数据，但是其中最重要的是集群健康指标，它会以green，yellow和red来报告集群的健康状态。

```
# Retrieve the cluster health
GET /_cluster/health
```

当集群中没有任何索引时，它会返回如下信息：

```json
{
   "cluster_name": "elasticsearch",
   "status": "green",
   "timed_out": false,
   "number_of_nodes": 1,
   "number_of_data_nodes": 1,
   "active_primary_shards": 0,
   "active_shards": 0,
   "relocating_shards": 0,
   "initializing_shards": 0,
   "unassigned_shards": 0
}
```

status字段提供的值反应了集群整体的健康程度。它的值的意义如下：

- green：所有的主分片(Primary Shard)和副本分片(Replica Shard)都处于活动状态
- yellow：所有的主分片都处于活动状态，但是并不是所有的副本分片都处于活跃状态
- red：不是所有的主分片都处于活动状态

在本章剩下的部分中，我们会解释什么是主分片和副本分片，以及以上的几种颜色状态信息所带来的实际影响。

## 添加索引

为了向ES中添加数据，我们需要一个索引(Index) - 它是一个用来存储相关数据的地方。实际上，一个索引实际上只是一个"逻辑命名空间(Logical Namespace)"，用来指向一个或者多个物理分片(Physical Shard)。

一个分片就是底层的"工作单元(Worker Unit)"，它拥有索引中所有数据的一部分。在[分片](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/inside-a-shard.html)一章中，会对分片的工作方式进行详细说明，但是就现在而言，我们只需要知道一个分片就是一个Lucene的实例，一个分片本身就是一个完整的搜索引擎。我们的文档会被存储和索引在分片中，但是应用是不会直接和分片进行交互的。相反地，应用和索引进行交互。

ES通过分片将数据分布在集群中。可以将分片想象成数据的容器。文档会被存储在分片中，而分片则会被分配到集群中的节点中。随着集群的扩大和虽小，ES会自动地将分片在节点之间进行迁移，以保证集群能够保持一种平衡。

一个分片可以是主分片(Primary Shard)或者副本分片(Replica Shard)。索引中的每份文档都属于一个主分片，所以主分片的数量就决定了你的索引能够存储的最大数据量。

> 尽管在理论上，一个主分片能够容纳的最大数据量并没有限制，但是在实际生产中这个限制是存在的。分片的最大空间完全取决于你的用例：硬件条件，文档的大小和复杂度，如何索引和查询文档，以及期望的响应时间。

一个副本分片则只是一个主分片的拷贝。副本用来提供数据冗余，用来保护数据在发生硬件故障是不会丢失，同时也能够处理像搜索和获取文档这样的读请求。

主分片的数量在索引建立之初就会被确定下来，而副本分片的数量则可以在任何时候被更改。

让我们在当前只有一个节点的集群中创建一个新的blogs索引。默认情况下，索引会拥有5个主分片，但是为了演示，我们会让索引有3个主分片和1个副本分片(每个主分片都有1个副本分片)：

```json
PUT /blogs
{
   "settings" : {
      "number_of_shards" : 3,
      "number_of_replicas" : 1
   }
}
```

此时我们的集群就变成这样了：

![](http://img.blog.csdn.net/20161202131945301?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

这个时候我们如果检查集群的健康状态，会得到如下的结果：

```json
{
   "cluster_name":          "elasticsearch",
   "status":                "yellow", 
   "timed_out":             false,
   "number_of_nodes":       1,
   "number_of_data_nodes":  1,
   "active_primary_shards": 3,
   "active_shards":         3,
   "relocating_shards":     0,
   "initializing_shards":   0,
   "unassigned_shards":     3 
}
```

集群的健康状态变成了黄色，同时响应中还说明了有3个未被分配的分片。

黄色说明了所有的主分片都正在正常运行，处于活动状态 - 集群现在能够成功处理来自外部的请求 - 但是并不是所有的副本分片都处于活动状态。实际上，所有的3个副本分片目前都处于"未分配"的状态 - 它们不存在于任何节点上。这是因为将相同数据的拷贝存放在同一节点上是没有意义的。如果我们失去了该节点，那么我们会失去所有数据和它们的拷贝。

因此当前我们的集群能够正常工作，只不过抵御不了硬件故障带来的风险。
