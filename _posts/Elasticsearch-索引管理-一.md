---
title: '[Elasticsearch] 索引管理 (一)'
date: 2014-11-24 10:04:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 索引]
---

## 索引管理

本文翻译自Elasticsearch官方指南的[索引管理(Index Management)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/index-management.html)一章

我们已经了解了ES是如何在不需要任何复杂的计划和安装就能让我们很容易地开始开发一个新的应用的。但是，用不了多久你就会想要仔细调整索引和搜索过程来更好的适配你的用例。

几乎所有的定制都和索引(Index)以及其中的类型(Type)相关。本章我们就来讨论用于管理索引和类型映射的API，以及最重要的设置。

<!-- More -->

### 创建索引

到现在为止，我们已经通过索引一份文档来完成了新索引的创建。这个索引是使用默认的设置创建的，新的域通过动态映射(Dynamic Mapping)的方式被添加到了类型映射(Type Mapping)中。

现在我们对这个过程拥有更多的控制：我们需要确保索引被创建时拥有合适数量的主分片(Primary Shard)，并且在索引任何数据之前，我们需要设置好解析器(Analyzers)以及映射(Mappings)。

因此我们需要手动地去创建索引，将任何需要的设置和类型映射传入到请求正文中，就像下面这样：

```json
PUT /my_index
{
    "settings": { ... },
    "mappings": {
        "type_one": { ... },
        "type_two": { ... },
        ...
    }
}
```

事实上，如果你想阻止索引被自动创建，可以通过添加下面的设置到每个节点的config/elasticsearch.yml文件中：

```
action.auto_create_index: false
```

> 将来，我们会讨论如何使用[索引模板(Index Template)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/index-templates.html)来预先定义自动生成的索引。这个功能在索引日志数据的时候有用武之地：索引的名字中会包含日期，每天都有一个有着合适配置的索引被自动地生成。

### 删除索引

使用下面的请求完成索引的删除：

```
DELETE /my_index
```

你也可以删除多个索引：

```
DELETE /index_one,index_two
DELETE /index_*
```

你甚至还可以删除所有的索引：

```
DELETE /_all
```

### 索引设置

虽然索引的种种行为可以通过[索引模块的参考文档](http://www.elasticsearch.org/guide/en/elasticsearch/reference/1.4//index-modules.html)介绍的那样进行配置，但是……

> TIP
> 
> ES中提供了一些很好的默认值，只有当你知道它是干什么的，以及为什么要去修改它的时候再去修改。

两个最重要的设置：

#### number_of_shards

一个索引中含有的主分片(Primary Shard)的数量，默认值是5。在索引创建后这个值是不能被更改的。

#### number_of_replicas

每一个主分片关联的副本分片(Replica Shard)的数量，默认值是1。这个设置在任何时候都可以被修改。

比如，我们可以通过下面的请求创建一个小的索引 - 只有一个主分片 - 同时没有副本分片：

```json
PUT /my_temp_index
{
    "settings": {
        "number_of_shards" :   1,
        "number_of_replicas" : 0
    }
}
```

将来，我们可以动态地通过update-index-settings API完成对副本分片数量的修改：

```json
PUT /my_temp_index/_settings
{
    "number_of_replicas": 1
}
```

置解析器

第三个重要的索引设置就是解析(Analysis)，可以利用已经存在的解析器(Analyzer)进行配置，或者是为你的索引定制新的解析器。

在[解析和解析器](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/analysis-intro.html)中，我们介绍了一些内置的解析器，它们用来将全文字符串转换成适合搜索的倒排索引(Inverted Index)。

对于全文字符串字段默认使用的是standard解析器，它对于多数西方语言而言是一个不错的选择。它包括：

- standard分词器。它根据词语的边界进行分词。
- standard token过滤器。用来整理上一步分词器得到的tokens，但是目前是一个空操作(no-op)。
- lowercase token过滤器。将所有tokens转换为小写。
- stop token过滤器。移除所有的stopwords，比如a，the，and，is等

默认下stopwords过滤器没有被使用。可以通过创建一个基于standard解析器的解析器并设置stopwords参数来启用。要么提供一个stopwords的列表或者告诉它使用针对某种语言预先定义的stopwords列表。

在下面的例子中，我们创建了一个名为es_std的解析器，它使用了预先定义的西班牙语中的stopwords列表：

```json
PUT /spanish_docs
{
    "settings": {
        "analysis": {
            "analyzer": {
                "es_std": {
                    "type":      "standard",
                    "stopwords": "_spanish_"
                }
            }
        }
    }
}
```

es_std解析器不是全局的 - 它只作用于spanish_docs索引。可以通过制定索引名，使用analyze API进行测试：

```json
GET /spanish_docs/_analyze?analyzer=es_std
{
    El veloz zorro marrón
}
```

下面的部分结果显示了西班牙语中的stopword El已经被正确地移除了：

```json
{
  "tokens" : [
    { "token" :    "veloz",   "position" : 2 },
    { "token" :    "zorro",   "position" : 3 },
    { "token" :    "marrón",  "position" : 4 }
  ]
}
```


