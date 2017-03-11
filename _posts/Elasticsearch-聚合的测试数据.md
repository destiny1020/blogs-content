---
title: '[Elasticsearch] 聚合的测试数据'
date: 2015-01-05 00:02:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 聚合]
---

本章翻译自Elasticsearch官方指南的[Aggregation Test-Drive](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/_aggregation_test_drive.html)一章。

# 聚合的测试数据(Aggregation Test-Drive)

我们将学习各种聚合以及它们的语法，但是最好的学习方法还是通过例子。一旦你了解了如何思考聚合以及如何对它们进行合适的嵌套，那么语法本身是不难的。

让我们从一个例子开始。我们会建立一个也许对汽车交易商有所用处的聚合。数据是关于汽车交易的：汽车型号，制造商，销售价格，销售时间以及一些其他的相关数据。

首先，通过批量索引(Bulk-Index)来添加一些数据：

<!-- More -->

```json
POST /cars/transactions/_bulk
{ "index": {}}
{ "price" : 10000, "color" : "red", "make" : "honda", "sold" : "2014-10-28" }
{ "index": {}}
{ "price" : 20000, "color" : "red", "make" : "honda", "sold" : "2014-11-05" }
{ "index": {}}
{ "price" : 30000, "color" : "green", "make" : "ford", "sold" : "2014-05-18" }
{ "index": {}}
{ "price" : 15000, "color" : "blue", "make" : "toyota", "sold" : "2014-07-02" }
{ "index": {}}
{ "price" : 12000, "color" : "green", "make" : "toyota", "sold" : "2014-08-19" }
{ "index": {}}
{ "price" : 20000, "color" : "red", "make" : "honda", "sold" : "2014-11-05" }
{ "index": {}}
{ "price" : 80000, "color" : "red", "make" : "bmw", "sold" : "2014-01-01" }
{ "index": {}}
{ "price" : 25000, "color" : "blue", "make" : "ford", "sold" : "2014-02-12" }
```

现在我们有了一些数据，来创建一个聚合吧。一个汽车交易商也许希望知道哪种颜色的车卖的最好。这可以通过一个简单的聚合完成。使用terms桶：

```json
GET /cars/transactions/_search?search_type=count 
{
    "aggs" : { 
        "colors" : { 
            "terms" : {
              "field" : "color" 
            }
        }
    }
}
```

因为我们并不关心搜索结果，使用的search_type是count，它的速度更快。 聚合工作在顶层的aggs参数下(当然你也可以使用更长的aggregations)。 然后给这个聚合起了一个名字：colors。 最后，我们定义了一个terms类型的桶，它针对color字段。

聚合是以搜索结果为上下文而执行的，这意味着它是搜索请求(比如，使用/_search端点)中的另一个顶层参数(Top-level Parameter)。聚合可以和查询同时使用，这一点我们在后续的[范围聚合(Scoping Aggregations)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/_scoping_aggregations.html)中介绍。

接下来我们为聚合起一个名字。命名规则是有你决定的; 聚合的响应会被该名字标记，因此在应用中你就能够根据名字来得到聚合结果，并对它们进行操作了。

然后，我们开始定义聚合本身。比如，我们定义了一个terms类型的桶。terms桶会动态地为每一个它遇到的不重复的词条创建一个新的桶。因为我们针对的是color字段，那么terms桶会动态地为每种颜色创建一个新桶。

让我们执行该聚合来看看其结果：

```json
{
   "hits": {
      "hits": [] 
   },
   "aggregations": {
      "colors": { 
         "buckets": [
            {
               "key": "red", 
               "doc_count": 4 
            },
            {
               "key": "blue",
               "doc_count": 2
            },
            {
               "key": "green",
               "doc_count": 2
            }
         ]
      }
   }
}
```

因为我们使用的search_type为count，所以没有搜索结果被返回。 每个桶中的key对应的是在color字段中找到的不重复的词条。它同时也包含了一个doc_count，用来表示包含了该词条的文档数量。

响应包含了一个桶列表，每个桶都对应着一个不重复的颜色(比如，红色或者绿色)。每个桶也包含了“掉入”该桶中的文档数量。比如，有4辆红色的车。

前面的例子是完全实时(Real-Time)的：如果文档是可搜索的，那么它们就能够被聚合。这意味着你能够将拿到的聚合结果置入到一个图形库中来生成实时的仪表板(Dashboard)。一旦你卖出了一台银色汽车，在图形上关于银色汽车的统计数据就会被动态地更新。

瞧！你的第一个聚合！

## 添加一个指标(Metric)

从前面的例子中，我们可以知道每个桶中的文档数量。但是，通常我们的应用会需要基于那些文档的更加复杂的指标(Metric)。比如，每个桶中的汽车的平均价格是多少？

为了得到该信息，我们得告诉ES需要为哪些字段计算哪些指标。这需要将指标嵌套到桶中。指标会基于桶中的文档的值来计算相应的统计信息。

让我们添加一个计算平均值的指标：

```json
GET /cars/transactions/_search?search_type=count
{
   "aggs": {
      "colors": {
         "terms": {
            "field": "color"
         },
         "aggs": { 
            "avg_price": { 
               "avg": {
                  "field": "price" 
               }
            }
         }
      }
   }
}
```

我们添加了一个新的aggs层级来包含该指标。然后给该指标起了一个名字：avg_price。最后定义了该指标作用的字段为price。.

正如你所看到的，我们向前面的例子中添加了一个新的aggs层级。这个新的聚合层级能够让我们将avg指标嵌套在terms桶中。这意味着我们能为每种颜色都计算一个平均值。

同样的，我们需要给指标起一个名(avg_price)来让我们能够在将来得到其值。最后，我们指定了指标本身(avg)以及该指标作用的字段(price)：

```json
{
   "aggregations": {
      "colors": {
         "buckets": [
            {
               "key": "red",
               "doc_count": 4,
               "avg_price": { 
                  "value": 32500
               }
            },
            {
               "key": "blue",
               "doc_count": 2,
               "avg_price": {
                  "value": 20000
               }
            },
            {
               "key": "green",
               "doc_count": 2,
               "avg_price": {
                  "value": 21000
               }
            }
         ]
      }
   }
}
```

现在，在响应中多了一个avg_price元素。

尽管得到的响应只是稍稍有些变化，但是获得的数据增加的了许多。之前我们只知道有4辆红色汽车。现在我们知道了红色汽车的平均价格是32500刀。这些数据你可以直接插入到报表中。

## 桶中的桶(Buckets inside Buckets)

当你开始使用不同的嵌套模式时，聚合强大的能力才会显现出来。在前面的例子中，我们已经知道了如何将一个指标嵌套进一个桶的，它的功能已经十分强大了。

但是真正激动人心的分析功能来源于嵌套在其它桶中的桶。现在，让我们来看看如何找到每种颜色的汽车的制造商分布信息：

```json
GET /cars/transactions/_search?search_type=count
{
   "aggs": {
      "colors": {
         "terms": {
            "field": "color"
         },
         "aggs": {
            "avg_price": { 
               "avg": {
                  "field": "price"
               }
            },
            "make": { 
                "terms": {
                    "field": "make" 
                }
            }
         }
      }
   }
}
```

此时发生了一些有意思的事情。首先，你会注意到前面的avg_price指标完全没有变化。一个聚合的每个层级都能够拥有多个指标或者桶。avg_price指标告诉了我们每种汽车颜色的平均价格。为每种颜色创建的桶和指标是各自独立的。

这个性质对你的应用而言是很重要的，因为你经常需要收集一些互相关联却又完全不同的指标。聚合能够让你对数据遍历一次就得到所有需要的信息。

另外一件重要的事情是添加了新聚合make，它是一个terms类型的桶(嵌套在名为colors的terms桶中)。这意味着我们会根据数据集创建不重复的(color, make)组合。

让我们来看看得到的响应(有省略，因为响应太长了)：

```json
{
   "aggregations": {
      "colors": {
         "buckets": [
            {
               "key": "red",
               "doc_count": 4,
               "make": { 
                  "buckets": [
                     {
                        "key": "honda", 
                        "doc_count": 3
                     },
                     {
                        "key": "bmw",
                        "doc_count": 1
                     }
                  ]
               },
               "avg_price": {
                  "value": 32500 
               }
            },
	...
}
```

该响应告诉了我们如下信息：

- 有4辆红色汽车。
- 红色汽车的平均价格是32500美刀。
- 红色汽车中的3辆是Honda，1辆是BMW。

## 最后的一个修改(One Final Modification)

在继续讨论新的话题前，为了把问题讲清楚让我们对该例子进行最后一个修改。为每个制造商添加两个指标来计算最低和最高价格：

```json
GET /cars/transactions/_search?search_type=count
{
   "aggs": {
      "colors": {
         "terms": {
            "field": "color"
         },
         "aggs": {
            "avg_price": { "avg": { "field": "price" }
            },
            "make" : {
                "terms" : {
                    "field" : "make"
                },
                "aggs" : { 
                    "min_price" : { "min": { "field": "price"} }, 
                    "max_price" : { "max": { "field": "price"} } 
                }
            }
         }
      }
   }
}
```

我们需要添加另一个aggs层级来进行对min和max的嵌套。

得到的响应如下(仍然有省略)：

```json
{
...
   "aggregations": {
      "colors": {
         "buckets": [
            {
               "key": "red",
               "doc_count": 4,
               "make": {
                  "buckets": [
                     {
                        "key": "honda",
                        "doc_count": 3,
                        "min_price": {
                           "value": 10000 
                        },
                        "max_price": {
                           "value": 20000 
                        }
                     },
                     {
                        "key": "bmw",
                        "doc_count": 1,
                        "min_price": {
                           "value": 80000
                        },
                        "max_price": {
                           "value": 80000
                        }
                     }
                  ]
               },
               "avg_price": {
                  "value": 32500
               }
            },
...
```

在每个make桶下，多了min和max的指标。

此时，我们可以得到如下信息：

- 有4辆红色汽车。
- 红色汽车的平均价格是32500美刀。
- 红色汽车中的3辆是Honda，1辆是BMW。
- 红色Honda汽车中，最便宜的价格为10000美刀。
- 最贵的红色Honda汽车为20000美刀。
