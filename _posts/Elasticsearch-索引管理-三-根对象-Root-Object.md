---
title: '[Elasticsearch] 索引管理 (三) - 根对象(Root Object)'
date: 2014-11-26 10:06:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 索引, Lucene]
---

## 根对象(Root Object)

映射的最顶层被称为根对象。它包含了：

- 属性区域(Properties Section)，列举了文档中包含的每个字段的映射信息。
- 各种元数据(Metadata)字段，它们都以_开头，比如_type，_id，_source。
- 控制用于新字段的动态探测(Dynamic Detection)的设置，如analyzer，dynamic_date_formats和dynamic_templates。
- 其它的可以用在根对象和object类型中的字段上的设置，如enabled，dynamic和include_in_all。

<!-- More -->

### 属性(Properties)

我们已经在[核心简单字段类型(Core Simple Field Type)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/mapping-intro.html#core-fields)和[复杂核心字段类型(Complex Core Field Type)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/complex-core-fields.html)中讨论了对于文档字段或属性最为重要的三个设置：

- type：字段的数据类型，比如string或者date。
- index：一个字段是否需要被当做全文(Full text)进行搜索(analyzed)，被当做精确值(Exact value)进行搜索('not_analyzed')，或者不能被搜索(no)。
- analyzer：全文字段在索引时(Index time)和搜索时(Search time)使用的analyzer。
我们会在后续章节中合适的地方讨论诸如ip，geo_point和geo_shape等其它字段类型。

### 元数据：_source字段

默认，ES会将表示文档正文的JSON字符串保存为_source字段。和其它存储的字段一样，_source字段也会在保存到磁盘上之前被压缩。

这个功能几乎是总被需要的，因为它意味着：

- 完整的文档在搜索结果中直接就是可用的 - 不需要额外的请求来得到完整文档
- _source字段让部分更新请求(Partial Update Request)成为可能
- 当映射发生变化而需要对数据进行重索引(Reindex)时，你可以直接在ES中完成，而不需要从另外一个数据存储(Datastore)(通常较慢)中获取所有文档
- 在你不需要查看整个文档时，可以从_source直接抽取出个别字段，通过get或者search请求返回
- 调试查询更容易，因为可以清楚地看到每个文档包含的内容，而不需要根据一个ID列表来对它们的内容进行猜测

即便如此，存储_store字段确实会占用磁盘空间。如果以上的任何好处对你都不重要，你可以使用以下的映射来禁用_source字段：

```json
PUT /my_index
{
    "mappings": {
        "my_type": {
            "_source": {
                "enabled":  false
            }
        }
    }
}
```

在一个搜索请求中，你可以只要求返回部分字段，通过在请求正文(Request body)中指定_source参数：

```json
GET /_search
{
    "query":   { "match_all": {}},
    "_source": [ "title", "created" ]
}
```

这些字段的值会从_source字段中被抽取出来并返回，而不是完整的_source。

> 存储字段(Stored fields)
> 
> 除了将一个字段的值索引外，你还可以选择将字段的原始值(Original field value)进行store来方便将来的获取。有过使用Lucene经验的用户会使用存储字段来选择在搜索结果中能够被返回的字段。实际上，_source字段就是一个存储字段。
> 
> 在ES中，设置个别的文档字段为存储字段通常都是一个错误的优化。整个文档已经通过_source字段被保存了。使用_source参数来指定需要抽取的字段几乎总是更好的方案。

### 元数据：_all字段

在[简化搜索(Search Lite)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/search-lite.html)中我们介绍了_all字段：它是一个特殊的字段，将其它所有字段的值当做一个大的字符串进行索引。query_string查询语句(以及?q=john这种形式的查询)在没有指定具体字段的时候，默认搜索的就是_all字段。

_all字段在一个新应用的探索阶段有用处，此时你对文档的最终结构还不太确定。你可以直接使用任何搜索字符串，并且也能够得到需要的结果：

```json
GET /_search
{
    "match": {
        "_all": "john smith marketing"
    }
}
```

随着你的应用逐渐成熟，对搜索要求也变的更加精确，你就会越来越少地使用_all字段。 _all字段是一种搜索的霰弹枪策略(Shotgun approach)。通过查询个别字段，你可以对搜索结果有更灵活，强大和细粒度的控制，来保证结果是最相关的。

> 在[相关度算法(Relevance Algorithm)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/relevance-intro.html)中一个重要的考量因素是字段的长度：字段越短，那么它就越重要。一个出现在较短的title字段中的词条会比它出现在较长的content字段中时要更重要。而这个关于字段长度的差别在_all字段中时不存在的。

如果你决定不再需要_all字段了，那么可以通过下面的映射设置来禁用它：

```json
PUT /my_index/_mapping/my_type
{
    "my_type": {
        "_all": { "enabled": false }
    }
}
```

可以使用include_in_all设置来对每个字段进行设置，是否需要将它包含到_all字段中，默认值是true。在一个对象(或者在根对象上)设置include_in_all会改变其中所有字段的默认设置。

如果你只需要将部分字段添加到_all字段中，比如title，overview，summary，tags等，用来方便地进行全文搜索。那么相比完全禁用_all，你可以将include_in_all默认设置为对所有字段禁用，然后对你选择的字段启用：

```json
PUT /my_index/my_type/_mapping
{
    "my_type": {
        "include_in_all": false,
        "properties": {
            "title": {
                "type":           "string",
                "include_in_all": true
            },
            ...
        }
    }
}
```

需要记住的是，_all字段也只不过是一个被解析过的string字段。它使用默认的解析器来解析其值，无论来源字段中设置的是什么解析器。和任何string字段一样，你也可以配置_all字段应该使用的解析器：

```json
PUT /my_index/my_type/_mapping
{
    "my_type": {
        "_all": { "analyzer": "whitespace" }
    }
}
```

### 元数据：文档ID(Document Identity)

和文档ID相关的有四个元数据字段：

- _id：文档的字符串ID
- _type：文档的类型
- _index：文档属于的索引
- _uid：_type和_id的结合，type#id
默认情况下，_uid字段会被保存和索引。意味着它可以被获取，也可以被搜索。_type字段会被索引但不会被保存。_id和_index既不会被索引也不会被保存，也就是说它们实际上是不存在的。

尽管如此，你还是能够查询_id字段，就好像它是一个实实在在的字段一样。ES使用_uid字段来得到_id。尽管你可以为这些字段修改index和store设置，但是你几乎不需要这么做。

_id字段有一个你也许会用到的设置：path，它用来告诉ES：文档应该从某个字段中抽取一个值来作为它自身的_id。

```json
PUT /my_index
{
    "mappings": {
        "my_type": {
            "_id": {
                "path": "doc_id" 
            },
            "properties": {
                "doc_id": {
                    "type":   "string",
                    "index":  "not_analyzed"
                }
            }
        }
    }
}
```

以上请求设置_id来源于doc_id字段。

然后，当你索引一份文档：

```json
POST /my_index/my_type
{
    "doc_id": "123"
}
```

得到的结果是这样的：

```json
{
    "_index":   "my_index",
    "_type":    "my_type",
    "_id":      "123", 
    "_version": 1,
    "created":  true
}
```

> 警告
> 
> 这样做虽然很方便，但是它对于bulk请求(参考[为什么选择有趣的格式](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/distrib-multi-doc.html#bulk-format))有一些性能影响。处理请求的节点不能够利用优化的批处理格式：仅通过解析元数据行来得知哪个分片(Shard)应该接受该请求。相反，它需要解析文档正文部分。
