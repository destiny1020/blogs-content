---
title: '[Elasticsearch] 分布式文档存储'
date: 2014-11-18 09:53:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 分布式, 存储]
---

本文翻译自Elasticsearch官方指南的[distributed document store](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/distributed-docs.html)一章。

## 分布式文档存储

在上一章中，我们一直在介绍索引数据和获取数据的方法。但是我们省略了很多关于数据是如何在集群中被分布(Distributed)和获取(Fetched)的技术细节。这实际上是有意为之 - 你真的不需要了解数据在ES中是如何被分布的。它能工作就足够了。

在本章中，我们将会深入到这些内部技术细节中，来帮助你了解你的数据是如何被存储在一个分布式系统中的。

<!-- More -->

### 路由一份文档(Document)到一个分片(Shard)

当你索引一份文档时，它会被保存到一个主要分片(Primary Shard)上。那么ES是如何知道该文档应该被保存到哪个分片上呢？当我们创建了一份新文档，ES是如何知道它究竟应该保存到分片1或者分片2上的呢？

这个过程不能是随机的，因为将来我们或许还需要获取该文档。实际上，这个过程是通过一个非常简单的公式决定的：

> shard = hash(routing) % number_of_primary_shards

以上的routing的值是一个任意的字符串，它默认被设置成文档的_id字段，但是也可以被设置成其他指定的值。这个routing字符串会被传入到一个哈希函数(Hash Function)来得到一个数字，然后该数字会和索引中的主要分片数进行模运算来得到余数。这个余数的范围应该总是在0和number_of_primary_shards - 1之间，它就是一份文档被存储到的分片的号码。

这就解释了为什么索引中的主要分片数量只能在索引创建时被指定，并且将来都不能在被更改：如果主要分片数量在索引创建后改变了，那么之前的所有路由结果都会变的不正确，从而导致文档不能被正确地获取。

> 用户有时会认为将主要分片数量固定下来会让将来对索引的水平扩展(Scale Out)变的困难。实际上，有些技术能够让你根据需要方便地进行水平扩展。我们会在[Designing for scale](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/scale.html)中介绍这些技术。

所有的文档API(get, index, delete, buli, update和mget)都接受一个routing参数，它用来定制从文档到分片的映射。一个特定的routing值能够确保所有相关文档 - 比如属于相同用户的所有文档 - 都会被存储在相同的分片上。我们会在[Designing for scale](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/scale.html)中详细介绍为什么你可能会这样做。

### 主要分片(Primary Shard)和副本分片(Replica Shard)是如何交互的

为了解释这个问题，假设我们有一个包含3个节点(Node)的集群(Cluster)。它含有一个拥有2个主要分片的名为blogs的索引。每个主要分片有2个副本分片。拥有相同数据的两个分片绝不会被分配到同一个节点上，所以这个集群的构成可能会像下图这样：

![](http://img.blog.csdn.net/20161202133031847?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

我们可以向集群中的任意一个节点发送请求。每个节点都有足够的能力来处理请求。每个节点都知道集群中的每份文档的位置，因此能够将请求转发到相应的节点。在下面的例子中，我们会将所有的请求都发送到节点1上，这个节点被称为请求节点(Requesting Node)。

TIP 当发送请求时，最好采用一种循环(Round-robin)的方式来将请求依次发送到每个节点上，从而做到分担负载。

### 文档的创建，索引和删除

创建，索引和删除的请求都是写操作(Write Operations)，它们都应该首先在主要分片(Primary Shard)上成功完成，然后才能被拷贝关联的副本分片(Replica Shard)上。

![](http://img.blog.csdn.net/20161202133046279?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

这个过程如上图所示。下面我们列举出图中用来完成创建，索引和删除文档的每个步骤：

1. 客户端(Client)向节点1发送了一个用于创建，索引或是删除的请求。
2. 节点使用该文档的_id字段来决定了它应该属于分片0。因此请求会被转发到节点3，因为分片0的主要分片目前被分配在节点3上。
3. 节点3会在文档对应的主要分片上执行这个请求。如果执行成功了，那么它会将该请求并行地转发到对应副本分片所在的节点1和节点2上。一旦所有的副本分片都成功完成了该请求，那么节点3就会向请求节点(Requesting Node)报告执行成功，然后节点3就能够向客户端发送一个请求执行成功的响应了。

当客户端接收到了执行成功的响应时，在主要分片和其关联的所有副本分片中，发送的文档已经被成功更新了。至此，你的修改就完成了。

在这个过程中还存在一些可选的参数用来对此过程进行调整，可能地如以牺牲数据安全的代价来增加性能。因为ES本身已经足够快了，所以这些可选参数很少被使用，但是为了完整性还是会对它们进行解释：

#### replication

replication的默认值是sync。它会导致主要分片将等待副本分片上的执行成功响应，然后才会将执行成功响应发送到请求节点。 如果你将replication设置成async，那么它会导致成功响应会在请求在主要分片上成功执行后就会被发送到客户端。它仍然会将请求转发到副本分片所在的节点上，只是你将无法得知请求在副本分片上是否能成功执行。 提到这个选项的目的是为了让你不要使用它。默认的sync值能够让ES处理各种系统中的数据压力。而使用了async可能会因为发送了过多无需等待其完成的请求而让ES处于过载的状态。

#### consistency

默认情况下，主要分片需要通过仲裁(Quorum)，即确认大部分分片拷贝(分片拷贝可以使主要分片或者副本分片，两者均可)有效时，才会发起一个写操作。这样做的目的是为了防止将数据写入到网络中"错误的一侧(Wrong Side)"。仲裁的定义如下：

> int( (primary + number_of_replicas) / 2 ) + 1

consistency的值可以是one(仅主要分片)，all(主要分片和所有副本分片)，或者是默认的quorum - 大部分分片拷贝。

注意number_of_replicas是指定在索引设置中的副本分片的数量，不是当前处于活动状态的副本分片数量。如果你在索引中指定了有3个副本分片的话，那么quorum的值就是：

> int( (primary + 3 replicas) / 2 ) + 1 = 3

那么当只启动了两个节点时，那么就无法满足quorum，从而导致无法索引或者删除任何文档。

timeout

如果没有足够的分片拷贝会如何呢？ES会等待，希望有更多的分片会出现。默认它会等待1分钟。如果需要可以将这个时间设置的短一些：100表示的是100毫秒，30s表示的是30秒。

> **NOTE** 
> 
> 一个新的索引默认会有1个副本分片，那么为了满足quorum则需要有两个活动的分片拷贝。但是，当ES运行在一个单一节点的集群上时，这些默认设置会阻止用户做任何有用的操作(比如索引等写操作)。为了防止这个问题，只有当number_of_replicas大于1时，quorum才需要被满足。

### 获取文档

文档能够通过主要分片(Primary Shard)或者任意一个副本分片(Replica Shard)获取。

![](http://img.blog.csdn.net/20161202133059388?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

上图展示了获取文档的过程，每个步骤解释如下：

1. 客户端(Client)发送一个请求到节点1。
2. 该节点利用文档的_id字段来判断该文档属于分片0。分片0的分片拷贝(主要分片或者是副本分片)存在于所有的3个节点上。这一次，它将请求转发到了节点2。
3. 节点2将文档返回给节点1，节点1随即将文档返回给客户端。

对于读请求(Read Request)，请求节点(Requesting Node)每次都会选择一个不同的分片拷贝来实现负载均衡 - 循环使用所有的分片拷贝。

可能存在这种情况，当一份文档正在被索引时，该文档在主要分片已经就绪了，但是还未被拷贝到其他副本分片上。此时副本分片或许报告文档不存在(译注：此时有读请求来获取该文档)，然而主要分片能够成功返回需要的文档 。一旦索引请求返回给用户的响应是成功，那么文档在主要分片以及所有副本分片上都是可用的。

### 文档的部分更新(Partial Update)

update API会结合读和写来完成一次部分更新。

![](http://img.blog.csdn.net/20161202133109754?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

下面对部分更新的步骤进行解释：

1. 客户端发送一个更新请求到节点1。
2. 节点1将请求转发到节点3，因为主要分片被分配在该节点上。
3. 节点3从主要分片中获取对应的文档，修改JSON文档中的_source字段，然后试图在该主要分片中对修改后的文档重新索引(Reindex)。如果该文档已经另外一个进程给修改了，那么它会根据retry_on_conflict设置的次数重试。
4. 如果节点3能够成功更新文档，它会将新版本的文档通过并行的方式给转发到副本分片所在的节点1和节点2上，也让它们进行重索引的操作。一旦所有的副本节点也执行成功，节点3就会将这一消息发送给请求节点(Requesting Node，此处就是节点1)。然后请求节点给客户端返回响应。

update API也能够接受接受routing，'replication'，'consistency'和'timeout'参数。

> 基于文档的复制
> 
> 当一个主要分片将修改转发给它的副本分片时，它不会转达更新请求。它转发的是新版本的完整文档。需要记住这些转发到副本分片的请求是异步的，也就是说它们到达的顺序和发送的顺序是不确定的。如果ES仅仅是转发修改，那么这些修改就可能以错误地顺序被接受，从而导致数据的损毁。

### 多文档模式(Multi-Document Patterns)

mget和bulk API的行为模式和单个的文档操作是类似的。区别主要在于请求节点知道每份文档被保存在那个分片上，因此就能够将一个多文档请求(Multi-Document Request)给拆分为针对每个分片的多文档请求，然后并行地将这些请求转发到对应的节点上。

一旦它从每个节点上获得了答案，它会将这些答案整理成一个单独的响应并返回给客户端。

![](http://img.blog.csdn.net/20161202133120732?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

使用一个mget请求来获取多份文档的步骤如下：

1. 客户端发送一个mget请求到节点1。
2. 节点1为每个分片(可以是主要分片或者副本分片)创建一个mget请求，然后将它们并行地转发到其他分片所在的节点。一旦节点1获得了所有的结果，就会将结果组装成响应最后返回给客户端。

每份文档都可以设置routing参数，通过传入一个docs数组来完成。

![](http://img.blog.csdn.net/20161202133130583?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

使用一个bulk请求来完成对多份文档的创建，索引，删除及更新的步骤如下：

1. 客户端发送一个bulk请求到节点1。
2. 节点1为每个分片(只能是主要分片)创建一个bulk请求，然后将它们并行地转发到其他主要分片所在的节点。
3. 主要分片会逐个执行bulk请求中出现的指令。当每个指令成功完成后，主要分片会将新的文档(或者删除的)并行地转发到它关联的所有副本分片上，然后执行下一条指令。一旦所有的副本分片对所有的指令都确定其成功了，那么当前节点就会向请求节点(Requesting Node)发送成功的响应，最后请求节点会整理所有的响应并最终发送响应给客户端。

bulk API也能够在整个请求的顶部接受replication和consistency参数，在每个具体的请求中接受routing参数。