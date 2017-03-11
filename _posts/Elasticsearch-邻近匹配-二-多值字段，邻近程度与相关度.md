---
title: '[Elasticsearch] 邻近匹配 (二) - 多值字段，邻近程度与相关度'
date: 2014-12-16 09:30:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 邻近匹配]
---

## 多值字段(Multivalue Fields)

在多值字段上使用短语匹配会产生古怪的行为：

```json
PUT /my_index/groups/1
{
    "names": [ "John Abraham", "Lincoln Smith"]
}
```

运行一个针对Abraham Lincoln的短语查询：

```json
GET /my_index/groups/_search
{
    "query": {
        "match_phrase": {
            "names": "Abraham Lincoln"
        }
    }
}
```

令人诧异的是，以上的这份文档匹配了查询。即使Abraham以及Lincoln分属于name数组的两个人名中。发生这个现象的原因在于数组在ES中的索引方式。

当John Abraham被解析时，它产生如下信息：

- 位置1：john
- 位置2：abraham

然后当Lincoln Smith被解析时，它产生了：

- 位置3：lincoln
- 位置4：smith

换言之，ES对以上数组分析产生的词条列表和解析单一字符串John Abraham Lincoln Smith时产生的结果是一样的。在我们的查询中，我们查询邻接的abraham和lincoln，而这两个词条在索引中确实存在并且邻接，因此查询匹配了。

幸运的是，有一个简单的方法来避免这种情况，通过position_offset_gap参数，它在字段映射中进行配置：

```json
DELETE /my_index/groups/ 

PUT /my_index/_mapping/groups 
{
    "properties": {
        "names": {
            "type":                "string",
            "position_offset_gap": 100
        }
    }
}
```

position_offset_gap设置告诉ES需要为数组中的每个新元素设置一个偏差值。因此，当我们再索引以上的人名数组时，会产生如下的结果：

- 位置1：john
- 位置2：abraham
- 位置103：lincoln
- 位置104：smith

现在我们的短语匹配就无法匹配该文档了，因为abraham和lincoln之间的距离为100。你必须要添加一个值为100的slop的值才能匹配。


## 越近越好(Closer is better)

短语查询(Phrase Query)只是简单地将不含有精确查询短语的文档排除在外，而邻近查询(Proximity Query) - 一个slop值大于0的短语查询 - 会将查询词条的邻近度也考虑到最终的相关度_score中。通过设置一个像50或100这样的高slop值，你可以排除那些单词过远的文档，但是也给予了那些单词邻近的文档一个更高的分值。

下面针对quick dog的邻近查询匹配了含有quick和dog的两份文档，但是给与了quick和dog更加邻近的文档一个更高的分值：

```json
POST /my_index/my_type/_search
{
   "query": {
      "match_phrase": {
         "title": {
            "query": "quick dog",
            "slop":  50 
         }
      }
   }
}
```

```json
{
  "hits": [
     {
        "_id":      "3",
        "_score":   0.75, 
        "_source": {
           "title": "The quick brown fox jumps over the quick dog"
        }
     },
     {
        "_id":      "2",
        "_score":   0.28347334, 
        "_source": {
           "title": "The quick brown fox jumps over the lazy dog"
        }
     }
  ]
}
```

## 使用邻近度来提高相关度

尽管邻近度查询(Proximity Query)管用，但是所有的词条都必须出现在文档的这一要求显的过于严格了。这个问题和我们在[全文搜索(Full-Text Search)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/full-text-search.html)一章的[精度控制(Controlling Precision)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/match-multi-word.html#match-precision)一节中讨论过的类似：如果7个词条中有6个匹配了，那么该文档也许对于用户而言已经足够相关了，但是match_phrase查询会将它排除在外。

相比将邻近度匹配作为一个绝对的要求，我们可以将它当做一个信号(Signal) - 作为众多潜在匹配中的一员，会对每份文档的最终分值作出贡献(参考[多数字段(Most Fields)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/most-fields.html))。

我们需要将多个查询的分值累加这一事实表示我们应该使用bool查询将它们合并。

我们可以使用一个简单的match查询作为一个must子句。该查询用于决定哪些文档需要被包含到结果集中。可以通过minimum_should_match参数来去除长尾(Long tail)。然后我们以should子句的形式添加更多特定查询。每个匹配了should子句的文档都会增加其相关度。

```json
GET /my_index/my_type/_search
{
  "query": {
    "bool": {
      "must": {
        "match": { 
          "title": {
            "query":                "quick brown fox",
            "minimum_should_match": "30%"
          }
        }
      },
      "should": {
        "match_phrase": { 
          "title": {
            "query": "quick brown fox",
            "slop":  50
          }
        }
      }
    }
  }
}
```

毫无疑问我们可以向should子句中添加其它的查询，每个查询都用来增加特定类型的相关度。