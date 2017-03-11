---
title: '[Elasticsearch] 控制相关度 (三) - 通过查询结构调整相关度以及boosting查询'
date: 2014-12-25 01:10:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 相关度分值计算]
---

本章翻译自Elasticsearch官方指南的[Controlling Relevance](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/controlling-relevance.html)一章。

## 通过查询结构调整相关度

ES提供的查询DSL是相当灵活的。你可以通过将单独的查询子句在查询层次中上下移动来让它更重要/更不重要。比如，下面的查询：

> quick OR brown OR red OR fox

我们可以使用一个bool查询，对所有词条一视同仁：

```json
GET /_search
{
  "query": {
    "bool": {
      "should": [
        { "term": { "text": "quick" }},
        { "term": { "text": "brown" }},
        { "term": { "text": "red"   }},
        { "term": { "text": "fox"   }}
      ]
    }
  }
}
```

<!-- More -->

但是这个查询会给一份含有quick，red及brown的文档和一份含有quick，red及fox的文档完全相同的分数，然而在[合并查询(Combining Queries)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/bool-query.html)中，我们知道bool查询不仅能够决定一份文档是否匹配，同时也能够知道该文档的匹配程度。

下面是更好的查询方式：

```json
GET /_search
{
  "query": {
    "bool": {
      "should": [
        { "term": { "text": "quick" }},
        { "term": { "text": "fox"   }},
        {
          "bool": {
            "should": [
              { "term": { "text": "brown" }},
              { "term": { "text": "red"   }}
            ]
          }
        }
      ]
    }
  }
}
```

现在，red和brown会在同一层次上相互竞争，而quick，fox以及red或者brown则是在顶层上相互对象的词条。

我们已经讨论了match，multi_match，term，book以及dis_max是如何对相关度分值进行操作的。在本章的剩余部分，我们会讨论和相关度分值有关的另外三种查询：boosting查询，constant_score查询以及function_score查询。

## 不完全的不(Not Quite Not)

在互联网上搜索"苹果"也许会返回关于公司，水果或者各种食谱的结果。我们可以通过排除pie，tart，crumble和tree这类单词，结合bool查询中的must_not子句，将结果范围缩小到只剩苹果公司：

```json
GET /_search
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "text": "apple"
        }
      },
      "must_not": {
        "match": {
          "text": "pie tart fruit crumble tree"
        }
      }
    }
  }
}
```

但是有谁敢说排除了tree或者crumble不会将一份原本和苹果公司非常相关的文档也排除在外了呢？有时，must_not过于严格了。

### boosting查询

[boosting查询](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/not-quite-not.html#boosting-query)能够解决这个问题。它允许我们仍然将水果或者食谱相关的文档考虑在内，只是会降低它们的相关度 - 将它们的排序更靠后：

```json
GET /_search
{
  "query": {
    "boosting": {
      "positive": {
        "match": {
          "text": "apple"
        }
      },
      "negative": {
        "match": {
          "text": "pie tart fruit crumble tree"
        }
      },
      "negative_boost": 0.5
    }
  }
}
```

它接受一个positive查询和一个negative查询。只有匹配了positive查询的文档才会被包含到结果集中，但是同时匹配了negative查询的文档会被降低其相关度，通过将文档原本的_score和negative_boost参数进行相乘来得到新的_score。

因此，negative_boost参数必须小于1.0。在上面的例子中，任何包含了指定负面词条的文档的_score都会是其原本_score的一半。