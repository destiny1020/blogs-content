---
title: '[Elasticsearch] 控制相关度 (五) -
  function_score查询及field_value_factor，boost_mode，max_mode参数'
date: 2014-12-27 23:20:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 相关度分值计算]
---

本章翻译自Elasticsearch官方指南的[Controlling Relevance](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/controlling-relevance.html)一章。

## function_score查询

[function_score查询](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html)是处理分值计算过程的终极工具。它让你能够对所有匹配了主查询的每份文档调用一个函数来调整甚至是完全替换原来的_score。

实际上，你可以通过设置过滤器来将查询得到的结果分成若干个子集，然后对每个子集使用不同的函数。这样你就能够同时得益于：高效的分值计算以及可缓存的过滤器。

它拥有几种预先定义好了的函数：

<!-- More -->

**weight**

对每份文档适用一个简单的提升，且该提升不会被归约：当weight为2时，结果为2 * _score。

**field_value_factor**

使用文档中某个字段的值来改变_score，比如将受欢迎程度或者投票数量考虑在内。

**random_score**

使用一致性随机分值计算来对每个用户采用不同的结果排序方式，对相同用户仍然使用相同的排序方式。

**衰减函数(Decay Function) - linear，exp，gauss**

将像publish_date，geo_location或者price这类浮动值考虑到_score中，偏好最近发布的文档，邻近于某个地理位置(译注：其中的某个字段)的文档或者价格(译注：其中的某个字段)靠近某一点的文档。

**script_score**

使用自定义的脚本来完全控制分值计算逻辑。如果你需要以上预定义函数之外的功能，可以根据需要通过脚本进行实现。

没有function_score查询的话，我们也许就不能将全文搜索得到分值和近因进行结合了。我们将不得不根据_score或者date进行排序；无论采用哪一种都会抹去另一种的影响。function_score查询让我们能够将两者融合在一起：仍然通过全文相关度排序，但是给新近发布的文档，或者流行的文档，或者符合用户价格期望的文档额外的权重。你可以想象，一个拥有所有这些功能的查询看起来会相当复杂。我们从一个简单的例子开始，循序渐进地对它进行介绍。

## 根据人气来提升(Boosting by Popularity)

假设我们有一个博客网站让用户投票选择他们喜欢的文章。我们希望让人气高的文章出现在结果列表的头部，但是主要的排序依据仍然是全文搜索分值。我们可以通过保存每篇文章的投票数量来实现：

```json
PUT /blogposts/post/1
{
  "title":   "About popularity",
  "content": "In this post we will talk about...",
  "votes":   6
}
```

在搜索期间，使用带有field_value_factor函数的function_score查询将投票数和全文相关度分值结合起来：

```json
GET /blogposts/post/_search
{
  "query": {
    "function_score": { 
      "query": { 
        "multi_match": {
          "query":    "popularity",
          "fields": [ "title", "content" ]
        }
      },
      "field_value_factor": { 
        "field": "votes" 
      }
    }
  }
}
```

function_score查询会包含主查询(Main Query)和希望适用的函数。先会执行主查询，然后再为匹配的文档调用相应的函数。每份文档中都必须有一个votes字段用来保证function_score能够起作用。

在前面的例子中，每份文档的最终_score会通过下面的方式改变：

> new_score = old_score * number_of_votes

它得到的结果并不好。全文搜索的_score通常会在0到10之间。而从下图我们可以发现，拥有10票的文章的分值大大超过了这个范围，而没有被投票的文章的分值会被重置为0。

![](https://camo.githubusercontent.com/f0bd555a4da2b95e912e92f4315de206eae017e0/687474703a2f2f7777772e656c61737469637365617263682e6f72672f67756964652f656e2f656c61737469637365617263682f67756964652f63757272656e742f696d616765732f656c61735f313730312e706e67)

**modifier**

为了让votes值对最终分值的影响更缓和，我们可以使用modifier。换言之，我们需要让头几票的效果更明显，其后的票的影响逐渐减小。0票和1票的区别应该比10票和11票的区别要大的多。

一个用于此场景的典型modifier是log1p，它将公式改成这样：

> new_score = old_score * log(1 + number_of_votes)

log函数将votes字段的效果减缓了，其效果类似下面的曲线：

![](https://camo.githubusercontent.com/2aaa20a0da983a8ccb8f1092ee302b57437bf045/687474703a2f2f7777772e656c61737469637365617263682e6f72672f67756964652f656e2f656c61737469637365617263682f67756964652f63757272656e742f696d616765732f656c61735f313730322e706e67)

使用了modifier参数的请求如下：

```json
GET /blogposts/post/_search
{
  "query": {
    "function_score": {
      "query": {
        "multi_match": {
          "query":    "popularity",
          "fields": [ "title", "content" ]
        }
      },
      "field_value_factor": {
        "field":    "votes",
        "modifier": "log1p" 
      }
    }
  }
}
```

可用的modifiers有：none(默认值)，log，log1p，log2p，ln，ln1p，ln2p，square，sqrt以及reciprocal。它们的详细功能和用法可以参考[field_value_factor文档](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html#_field_value_factor)。

**factor**

可以通过将votes字段的值乘以某个数值来增加该字段的影响力，这个数值被称为factor：

```json
GET /blogposts/post/_search
{
  "query": {
    "function_score": {
      "query": {
        "multi_match": {
          "query":    "popularity",
          "fields": [ "title", "content" ]
        }
      },
      "field_value_factor": {
        "field":    "votes",
        "modifier": "log1p",
        "factor":   2 
      }
    }
  }
}
```

添加了factor将公式修改成这样：

> new_score = old_score * log(1 + factor * number_of_votes)

当factor大于1时，会增加其影响力，而小于1的factor则相应减小了其影响力，如下图所示：

![](https://camo.githubusercontent.com/99c91b5d4e3d509d3c4981a88e446afc6c62320d/687474703a2f2f7777772e656c61737469637365617263682e6f72672f67756964652f656e2f656c61737469637365617263682f67756964652f63757272656e742f696d616765732f656c61735f313730332e706e67)

**boost_mode**

将全文搜索的相关度分值乘以field_value_factor函数的结果，对最终分值的影响可能太大了。通过boost_mode参数，我们可以控制函数的结果应该如何与_score结合在一起，该参数接受下面的值：

- multiply：_score乘以函数结果(默认情况)
- sum：_score加上函数结果
- min：_score和函数结果的较小值
- max：_score和函数结果的较大值
- replace：将_score替换成函数结果
如果我们是通过将函数结果累加来得到_score，其影响会小的多，特别是当我们使用了一个较低的factor时：

```json
GET /blogposts/post/_search
{
  "query": {
    "function_score": {
      "query": {
        "multi_match": {
          "query":    "popularity",
          "fields": [ "title", "content" ]
        }
      },
      "field_value_factor": {
        "field":    "votes",
        "modifier": "log1p",
        "factor":   0.1
      },
      "boost_mode": "sum" 
    }
  }
}
```

上述请求的公式如下所示：

> new_score = old_score + log(1 + 0.1 * number_of_votes)

![](https://camo.githubusercontent.com/38804d86f54890e2955a34c75775861d195c1ca3/687474703a2f2f7777772e656c61737469637365617263682e6f72672f67756964652f656e2f656c61737469637365617263682f67756964652f63757272656e742f696d616765732f656c61735f313730342e706e67)

**max_boost**

最后，我们能够通过制定max_boost参数来限制函数的最大影响：

```json
GET /blogposts/post/_search
{
  "query": {
    "function_score": {
      "query": {
        "multi_match": {
          "query":    "popularity",
          "fields": [ "title", "content" ]
        }
      },
      "field_value_factor": {
        "field":    "votes",
        "modifier": "log1p",
        "factor":   0.1
      },
      "boost_mode": "sum",
      "max_boost":  1.5 
    }
  }
}
```

无论field_value_factor函数的结果是多少，它绝不会大于1.5。

> NOTE
> 
> max_boost只是对函数的结果有所限制，并不是最终的_score。
