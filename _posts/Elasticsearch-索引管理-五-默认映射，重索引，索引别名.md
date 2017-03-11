---
title: '[Elasticsearch] 索引管理 (五) - 默认映射，重索引，索引别名'
date: 2014-12-01 10:13:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 索引]
---

## 默认映射(Default Mapping)

一般情况下，索引中的所有类型都会有相似的字段和设置。因此将这些常用设置在_default映射中指定会更加方便，这样就不需要在每次创建新类型的时候都重复设置。_default映射的角色是新类型的模板。所有在_default映射之后创建的类型都会包含所有的默认设置，除非显式地在类型映射中进行覆盖。

<!-- More -->

比如，我们使用_default映射对所有类型禁用_all字段，唯独对blog类型启用它。可以这样实现：

```json
PUT /my_index
{
    "mappings": {
        "_default_": {
            "_all": { "enabled":  false }
        },
        "blog": {
            "_all": { "enabled":  true  }
        }
    }
}
```

_default_映射同时也是一个声明作用于整个索引的[动态模板(Dynamic Templates)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/custom-dynamic-mapping.html#dynamic-templates)的好地方。

## 数据重索引

虽然你可以向索引中添加新的类型，或者像类型中添加新的字段，但是你不能添加新的解析器或者对现有字段进行修改。如果你这么做了，就会让已经索引的数据变的不正确，导致搜索不能正常的进行。

为已经存在的数据适用这些更改的最简单的方法就是重索引(Reindex)：新建一个拥有最新配置的索引，然后将所有旧索引中的数据拷贝到新的索引中。

_source字段的一个优势是在ES中你已经拥有了整个文档。你不需要通过数据库来重建你的索引，这种方法通常会更慢。

为了从旧索引中高效地对所有文档进行重索引，可以使用[scan和scroll](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/scan-scroll.html)来批量地从旧索引中获取文档，然后使用[bulk API](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/bulk.html)将它们添加到新索引中。

> 批量重索引
> 
> 你可以同时执行多个重索引任务，但是你显然不想要它们的结果有任何重叠。可以根据date或者timestamp字段对重索引的任务进行划分，成为规模较小的任务：
> 
> ```json
> GET /old_index/_search?search_type=scan&scroll=1m
> {
>     "query": {
>         "range": {
>             "date": {
>                 "gte":  "2014-01-01",
>                 "lt":   "2014-02-01"
>             }
>         }
>     },
>     "size":  1000
> }
> ```
> 如果你正在持续地更改旧索引中的数据，想必你也希望这些更改也会被反映到新索引中。这可以通过再次运行重索引任务来完成，但是还是可以通过对日期字段进行过滤来得到在上次重索引开始后才被添加的文档。

## 索引别名和零停机时间(Index Alias and Zero Downtime)

重索引的问题在于你需要更新你的应用让它使用新的索引名。而索引别名可以解决这个问题。

一个索引别名就好比一个快捷方式(Shortcut)或一个符号链接(Symbolic Link)，索引别名可以指向一个或者多个索引，可以在任何需要索引名的API中使用。使用别名可以给我们非常多的灵活性。它能够让我们：

- 在一个运行的集群中透明地从一个索引切换到另一个索引
- 让多个索引形成一个组，比如last_three_months
- 为一个索引中的一部分文档创建一个视图(View)

我们会在本书的后面讨论更多关于别名的其它用途。现在我们要解释的是如何在零停机时间的前提下，使用别名来完成从旧索引切换到新索引。

有两个用来管理别名的端点(Endpoint)：_alias用来完成单一操作，_aliases用来原子地完成多个操作。

在这个场景中，我们假设你的应用正在使用一个名为my_index的索引。实际上，my_index是一个别名，它指向了当前正在使用的真实索引。我们会在真实索引的名字中包含一个版本号码：my_index_v1，my_index_v2等。

首先，创建索引my_index_v1，然后让别名my_index指向它：

```
PUT /my_index_v1 
PUT /my_index_v1/_alias/my_index 
```

可以通过下面的请求得到别名指向的索引：

```
GET /*/_alias/my_index
```

或者查询指向真实索引的有哪些别名：

```
GET /my_index_v1/_alias/*
```

它们都会返回：

```json
{
    "my_index_v1" : {
        "aliases" : {
            "my_index" : { }
        }
    }
}
```

然后，我们决定要为索引中的一个字段更改其映射。当然，我们是不能修改当前的映射的，因此我们只好对数据进行重索引。此时我们创建了拥有新的映射的索引my_index_v2：

```json
PUT /my_index_v2
{
    "mappings": {
        "my_type": {
            "properties": {
                "tags": {
                    "type":   "string",
                    "index":  "not_analyzed"
                }
            }
        }
    }
}
```

紧接着，我们会根据[数据重索引](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/reindex.html)中的流程将my_index_v1中的数据重索引到my_index_v2。一旦我们确定了文档已经被正确地索引，我们就能够将别名切换到新的索引上了。

一个别名能够指向多个索引，因此当我们将别名指向新的索引时，我们还需要删除别名原来到旧索引的指向。这个改变需要是原子的，即意味着我们需要使用_aliases端点：

```json
POST /_aliases
{
    "actions": [
        { "remove": { "index": "my_index_v1", "alias": "my_index" }},
        { "add":    { "index": "my_index_v2", "alias": "my_index" }}
    ]
}
```

现在你的应用就在零停机时间的前提下，实现了旧索引到新索引的透明切换。

> TIP
> 
> 即使你认为当前的索引设计是完美的，将来你也会发现有一些部分需要被改变，而那个时候你的索引已经在生产环境中被使用了。
> 
> 在应用中使用索引别名而不是索引的真实名称，这样你就能够在任何需要的时候执行重索引操作。应该充分地使用别名。