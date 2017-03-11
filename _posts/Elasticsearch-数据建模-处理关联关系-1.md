---
title: '[Elasticsearch] 数据建模 - 处理关联关系(1)'
date: 2015-08-16 23:55:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 设计]
---

# 数据建模(Modeling Your Data)

ES是一头不同寻常的野兽，尤其是当你来自SQL的世界时。它拥有很多优势：性能，可扩展性，准实时的搜索，以及对大数据的分析能力。并且，它很容易上手！只需要下载就能够开始使用它了。

但是它也不是魔法。为了更好的利用ES，你需要了解它从而让它能够满足你的需求。

在ES中，处理实体之间的关系并不像关系型存储那样明显。在关系数据库中的黄金准则 - 数据规范化，在ES中并不适用。在[处理关联关系](https://www.elastic.co/guide/en/elasticsearch/guide/current/relations.html)，[嵌套对象](https://www.elastic.co/guide/en/elasticsearch/guide/current/nested-objects.html)和[父子关联关系](https://www.elastic.co/guide/en/elasticsearch/guide/current/parent-child.html)中，我们会讨论几种可行方案的优点和缺点。

<!-- More -->

紧接着在[为可扩展性而设计](https://www.elastic.co/guide/en/elasticsearch/guide/current/scale.html)中，我们会讨论ES提供的一些用来快速灵活实现扩展的特性。对于扩展，并没有一个可以适用于所有场景的解决方案。你需要考虑数据是如何在你的系统中流转的，从而恰当地对你的数据进行建模。针对基于时间的数据比如日志事件或者社交数据流的方案比相对静态的文档集合的方案是十分不同的。

最后，我们会讨论一样在ES中不会扩展的东西。

---

# 处理关联关系(Handling Relationships)

在真实的世界中，关联关系很重要：博客文章有评论，银行账户有交易，客户有银行账户，订单有行项目，目录也拥有文件和子目录。

在关系数据库中，处理关联关系的方式让你不会感到意外：

- 每个实体(或者行，在关系世界中)可以通过一个主键唯一标识。
- 实体是规范化了的。对于一个唯一的实体，它的数据仅被存储一次，而与之关联的实体则仅仅保存它的主键。改变一个实体的数据只能发生在一个地方。
- 在查询期间，实体可以被联接(Join)，它让跨实体查询成为可能。
- 对于单个实体的修改是原子性，一致性，隔离性和持久性的。(参考[ACID事务](http://en.wikipedia.org/wiki/ACID_transactions)获取更多相关信息。)
- 绝大多数的关系型数据库都支持针对多个实体的ACID事务。

但是关系型数据库也有它们的局限，除了在全文搜索领域它们拙劣的表现外。在查询期间联接实体是昂贵的 - 联接的实体越多，那么查询的代价就越大。对不同硬件上的实体执行联接操作的代价太大以至于它甚至是不切实际的。这就为在单个服务器上能够存储的数据量设下了一个限制。

ES，像多数NoSQL数据库那样，将世界看作是平的。一个索引就是一系列独立文档的扁平集合。一个单一的文档应该包括用来判断它是否符合一个搜索请求的所有信息。

虽然在ES中改变一份文档的数据是符合[ACIDic](http://en.wikipedia.org/wiki/ACID_transactions)的，涉及到多份文档的事务就不然了。在ES中，当事务失败后是没有办法将索引回滚到它之前的状态的。

这个扁平化的世界有它的优势：

- 索引是迅速且不需要上锁的。
- 搜索是迅速且不需要上锁的。
- 大规模的数据可以被分布到多个节点上，因为每份文档之间是独立的。

但是关联关系很重要。我们需要以某种方式将扁平化的世界和真实的世界连接起来。在ES中，有4中常用的技术来管理关联数据：

- [应用端联接(Application-side joins)](https://www.elastic.co/guide/en/elasticsearch/guide/current/application-joins.html)
- [数据非规范化(Data denormalization)](https://www.elastic.co/guide/en/elasticsearch/guide/current/denormalization.html)
- [嵌套对象(Nested objects)](https://www.elastic.co/guide/en/elasticsearch/guide/current/nested-objects.html)
- [父子关联关系(Parent/child relationships)](https://www.elastic.co/guide/en/elasticsearch/guide/current/parent-child.html)

通常最终的解决方案会结合这些方案的几种。

---

## 应用端联接(Application-side Joins)

我们可以通过在应用中实现联接来(部分)模拟一个关系型数据库。比如，当我们想要索引用户和他们的博客文章时。在关系型的世界中，我们可以这样做：

```json
PUT /my_index/user/1  (1)
{
  "name":     "John Smith",
  "email":    "john@smith.com",
  "dob":      "1970/10/24"
}

PUT /my_index/blogpost/2   (2)
{
  "title":    "Relationships",
  "body":     "It's complicated...",
  "user":     1   (3)
}
```

(1)(2) 索引，类型以及每份文档的ID一起构成了主键。

(3) 博文通过保存了用户的ID来联接到用户。由于索引和类型是被硬编码到了应用中的，所以这里并不需要。

通过用户ID等于1来找到对应的博文很容易：

```json
GET /my_index/blogpost/_search
{
  "query": {
    "filtered": {
      "filter": {
        "term": { "user": 1 }
      }
    }
  }
}
```

为了找到用户John的博文，我们可以执行两条查询：第一条查询用来得到所有名为John的用户的IDs，第二条查询通过这些IDs来得到对应文章：

```json
GET /my_index/user/_search
{
  "query": {
    "match": {
      "name": "John"
    }
  }
}

GET /my_index/blogpost/_search
{
  "query": {
    "filtered": {
      "filter": {
        "terms": { "user": [1] }   (1)
      }
    }
  }
}
```

(1) 传入到terms过滤器的值是第一条查询的结果。

应用端联接最大的优势在于数据是规范化了的。改变用户的名字只需要在一个地方操作：用户对应的文档。劣势在于你需要在搜索期间运行额外的查询来联接文档。

在这个例子中，只有一位用户匹配了第一条查询，但是在实际应用中可能轻易就得到了数以百万计的名为John的用户。将所有的IDs传入到第二个查询中会让该查询非常巨大，它需要执行百万计的term查询。

这种方法在第一个实体的文档数量较小并且它们很少改变时合适(这个例子中实体指的是用户)。这就使得通过缓存结果来避免频繁查询成为可能。

---

## 反规范化你的数据(Denormalizing Your Data)

让ES达到最好的搜索性能的方法是采用更直接的办法，通过在索引期间[反规范化](http://en.wikipedia.org/wiki/Denormalization)你的数据。通过在每份文档中包含冗余数据来避免联接。

如果我们需要通过作者的名字来搜索博文，可以在博文对应的文档中直接包含该作者的名字：

```json
PUT /my_index/user/1
{
  "name":     "John Smith",
  "email":    "john@smith.com",
  "dob":      "1970/10/24"
}

PUT /my_index/blogpost/2
{
  "title":    "Relationships",
  "body":     "It's complicated...",
  "user":     {
    "id":       1,
    "name":     "John Smith" 
  }
}
```

现在，我们可以通过一条查询来得到用户名为John的博文了：

```json
GET /my_index/blogpost/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title":     "relationships" }},
        { "match": { "user.name": "John"          }}
      ]
    }
  }
}
```

对数据的反规范化的优势在于速度。因为每份文档包含了用于判断是否匹配查询的所有数据，不需要执行代价高昂的联接操作。
