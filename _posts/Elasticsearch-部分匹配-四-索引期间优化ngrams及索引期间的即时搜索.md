---
title: '[Elasticsearch] 部分匹配 (四) - 索引期间优化ngrams及索引期间的即时搜索'
date: 2014-12-22 09:25:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 部分匹配]
---

本章翻译自Elasticsearch官方指南的[Partial Matching](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/partial-matching.html)一章。

## 索引期间的优化(Index-time Optimizations)

目前我们讨论的所有方案都是在查询期间的。它们不需要任何特殊的映射或者索引模式(Indexing Patterns)；它们只是简单地工作在已经存在于索引中的数据之上。

查询期间的灵活性是有代价的：搜索性能。有时，将这些代价放到查询之外的地方是有价值的。在一个实时的Web应用中，一个额外的100毫秒的延迟会难以承受。

通过在索引期间准备你的数据，可以让你的搜索更加灵活并更具效率。你仍然付出了代价：增加了的索引大小和稍微低一些的索引吞吐量，但是这个代价是在索引期间付出的，而不是在每个查询的执行期间。

你的用户会感激你的。

<!-- More -->

## 部分匹配(Partial Matching)的ngrams

我们说过："你只能找到存在于倒排索引中的词条"。尽管prefix，wildcard以及regexp查询证明了上面的说法并不是一定正确，但是执行一个基于单个词条的查询会比遍历词条列表来得到匹配的词条要更快是毫无疑问的。为了部分匹配而提前准备你的数据能够增加搜索性能。

在索引期间准别数据意味着选择正确的分析链(Analysis Chain)，为了部分匹配我们选择的工具叫做n-gram。一个n-gram可以被想象成一个单词上的滑动窗口(Moving Window)。n表示的是长度。如果我们对单词quick得到n-gram，结果取决于选择的长度：

- 长度1(unigram)： [ q, u, i, c, k ]
- 长度2(bigram)： [ qu, ui, ic, ck ]
- 长度3(trigram)： [ qui, uic, ick ]
- 长度4(four-gram)：[ quic, uick ]
- 长度5(five-gram)：[ quick ]

单纯的n-grams对于匹配单词中的某一部分是有用的，在[复合单词的ngrams](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/ngrams-compound-words.html)中我们会用到它。然而，对于即时搜索，我们使用了一种特殊的n-grams，被称为边缘n-grams(Edge n-grams)。边缘n-grams会将起始点放在单词的开头处。单词quick的边缘n-gram如下所示：

- q
- qu
- qui
- quic
- quick

你也许注意到它遵循了用户在搜索"quick"时的输入形式。换言之，对于即时搜索而言它们是非常完美的词条。

## 索引期间的即时搜索(Index-time Search-as-you-type)

建立索引期间即时搜索的第一步就是定义你的分析链(Analysis Chain)(在[配置解析器](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/configuring-analyzers.html)中讨论过)，在这里我们会详细阐述这些步骤：

### 准备索引

第一步是配置一个自定义的edge_ngram词条过滤器，我们将它称为autocomplete_filter：

```json
{
    "filter": {
        "autocomplete_filter": {
            "type":     "edge_ngram",
            "min_gram": 1,
            "max_gram": 20
        }
    }
}
```

以上配置的作用是，对于此词条过滤器接受的任何词条，它都会产生一个最小长度为1，最大长度为20的边缘ngram(Edge ngram)。

然后我们将该词条过滤器配置在自定义的解析器中，该解析器名为autocomplete。

```json
{
    "analyzer": {
        "autocomplete": {
            "type":      "custom",
            "tokenizer": "standard",
            "filter": [
                "lowercase",
                "autocomplete_filter" 
            ]
        }
    }
}
```

以上的解析器会使用standard分词器将字符串划分为独立的词条，将它们变成小写形式，然后为它们生成边缘ngrams，这要感谢autocomplete_filter。

创建索引，词条过滤器和解析器的完整请求如下所示：

```json
PUT /my_index
{
    "settings": {
        "number_of_shards": 1, 
        "analysis": {
            "filter": {
                "autocomplete_filter": { 
                    "type":     "edge_ngram",
                    "min_gram": 1,
                    "max_gram": 20
                }
            },
            "analyzer": {
                "autocomplete": {
                    "type":      "custom",
                    "tokenizer": "standard",
                    "filter": [
                        "lowercase",
                        "autocomplete_filter" 
                    ]
                }
            }
        }
    }
}
```

你可以通过下面的analyze API来确保行为是正确的：

```
GET /my_index/_analyze?analyzer=autocomplete
quick brown
```

返回的词条说明解析器工作正常：

- q
- qu
- qui
- quic
- quick
- b
- br
- bro
- brow
- brown

为了使用它，我们需要将它适用到字段中，通过update-mapping API：

```json
PUT /my_index/_mapping/my_type
{
    "my_type": {
        "properties": {
            "name": {
                "type":     "string",
                "analyzer": "autocomplete"
            }
        }
    }
}
```

现在，让我们索引一些测试文档：

```json
POST /my_index/my_type/_bulk
{ "index": { "_id": 1            }}
{ "name": "Brown foxes"    }
{ "index": { "_id": 2            }}
{ "name": "Yellow furballs" }
```

### 查询该字段

如果你使用一个针对"brown fo"的简单match查询：

```json
GET /my_index/my_type/_search
{
    "query": {
        "match": {
            "name": "brown fo"
        }
    }
}
```

你会发现两份文档都匹配了，即使Yellow furballs既不包含brown，也不包含fo：

```json
{

  "hits": [
     {
        "_id": "1",
        "_score": 1.5753809,
        "_source": {
           "name": "Brown foxes"
        }
     },
     {
        "_id": "2",
        "_score": 0.012520773,
        "_source": {
           "name": "Yellow furballs"
        }
     }
  ]
}
```

通过validate-query API来发现问题：

```json
GET /my_index/my_type/_validate/query?explain
{
    "query": {
        "match": {
            "name": "brown fo"
        }
    }
}
```

得到的解释说明了查询会寻找查询字符串中每个单词的边缘ngrams：

> name:b name:br name:bro name:brow name:brown name:f name:fo

name:f这一条件满足了第二份文档，因为furballs被索引为f，fu，fur等。因此，得到以上的结果也没什么奇怪的。autocomplete解析器被同时适用在了索引期间和搜索期间，通常而言这都是正确的行为。但是当前的场景是为数不多的不应该使用该规则的场景之一。

我们需要确保在倒排索引中含有每个单词的边缘ngrams，但是仅仅匹配用户输入的完整单词(brown和fo)。我们可以通过在索引期间使用autocomplete解析器，而在搜索期间使用standard解析器来达到这个目的。直接在查询中指定解析器就是一种改变搜索期间分析器的方法：

```json
GET /my_index/my_type/_search
{
    "query": {
        "match": {
            "name": {
                "query":    "brown fo",
                "analyzer": "standard" 
            }
        }
    }
}
```

另外，还可以在name字段的映射中分别指定index_analyzer和search_analyzer。因为我们只是想修改search_analyzer，所以可以在不对数据重索引的前提下对映射进行修改：

```json
PUT /my_index/my_type/_mapping
{
    "my_type": {
        "properties": {
            "name": {
                "type":            "string",
                "index_analyzer":  "autocomplete", 
                "search_analyzer": "standard" 
            }
        }
    }
}
```

此时再通过validate-query API得到的解释如下：

> name:brown name:fo

重复执行查询后，也仅仅会得到Brown foxes这份文档。

因为大部分的工作都在索引期间完成了，查询需要做的只是查找两个词条：brown和fo，这比使用match_phrase_prefix来寻找所有以fo开头的词条更加高效。

> 完成建议(Completion Suggester)
> 
> 使用边缘ngrams建立的即时搜索是简单，灵活和迅速的。然而，有些时候它还是不够快。延迟的影响不容忽略，特别当你需要提供实时反馈时。有时最快的搜索方式就是没有搜索。
> 
> ES中的完成建议采用了一种截然不同的解决方案。通过给它提供一个完整的可能完成列表(Possible Completions)来创建一个有限状态转换器(Finite State Transducer)，该转换器是一个用来描述图(Graph)的优化数据结构。为了搜索建议，ES会从图的起始处开始，对用户输入逐个字符地沿着匹配路径(Matching Path)移动。一旦用户输入被检验完毕，它就会根据当前的路径产生所有可能的建议。
> 
> 该数据结构存在于内存中，因此对前缀查询而言是非常迅速的，比任何基于词条的查询都要快。使用它来自动完成名字和品牌(Names and Brands)是一个很不错的选择，因为它们通常都以某个特定的顺序进行组织，比如"Johnny Rotten"不会被写成"Rotten Johnny"。
> 
> 当单词顺序不那么容易被预测时，边缘ngrams就是相比完成建议更好的方案。

### 边缘ngrams和邮政编码

边缘ngrams这一技术还可以被用在结构化数据上，比如本章[前面提到过的邮政编码](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/prefix-query.html)。当然，postcode字段也许需要被设置为analyzed，而不是not_analyzed，但是你仍然可以通过为邮政编码使用keyword分词器来让它们和not_analyzed字段一样。

> TIP
> 
> keyword分词器是一个没有任何行为(no-operation)的分词器。它接受的任何字符串会被原样输出为一个词条。所以对于一些通常被当做not_analyzed字段，然而需要某些处理(如转换为小写)的情况下，是有用处的。

这个例子使用keyword分词器将邮政编码字符串转换为一个字符流，因此我们就能够利用边缘ngram词条过滤器了：

```json
{
    "analysis": {
        "filter": {
            "postcode_filter": {
                "type":     "edge_ngram",
                "min_gram": 1,
                "max_gram": 8
            }
        },
        "analyzer": {
            "postcode_index": { 
                "tokenizer": "keyword",
                "filter":    [ "postcode_filter" ]
            },
            "postcode_search": { 
                "tokenizer": "keyword"
            }
        }
    }
}
```
