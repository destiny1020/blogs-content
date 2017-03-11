---
title: '[Elasticsearch] 邻近匹配 (三) - 性能，关联单词查询以及Shingles'
date: 2014-12-17 10:15:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 邻近匹配]
---

## 提高性能

短语和邻近度查询比简单的match查询在性能上更昂贵。match查询只是查看词条是否存在于倒排索引(Inverted Index)中，而match_phrase查询则需要计算和比较多个可能重复词条(Multiple possibly repeated)的位置。

在[Lucene Nightly Benchmarks](http://people.apache.org/~mikemccand/lucenebench/)中，显示了一个简单的term查询比一个短语查询快大概10倍，比一个邻近度查询(一个拥有slop的短语查询)快大概20倍。当然，这个代价是在搜索期间而不是索引期间付出的。

<!-- More -->

> TIP
> 
> 通常，短语查询的额外代价并不像这些数字说的那么吓人。实际上，性能上的差异只是说明了一个简单的term查询时多么的快。在标准全文数据上进行的短语查询通常能够在数毫秒内完成，因此它们在实际生产环境下是完全能够使用的，即使在一个繁忙的集群中。
> 
> 在某些特定的场景下，短语查询可能会很耗费资源，但是这种情况时不常有的。一个典型的例子是DNA序列，此时会在很多位置上出现非常之多的相同重复词条。使用高slop值会使位置计算发生大幅度的增长。

因此，如何能够限制短语和邻近度查询的性能消耗呢？一个有用的方法是减少需要使用短语查询进行检查的文档总数。

### 结果的分值重计算(Rescoring Results)

在上一节中，我们讨论了使用邻近度查询来调整相关度，而不是使用它来将文档从结果列表中添加或者排除。一个查询可能会匹配百万计的结果，但是我们的用户很可能只对前面几页结果有兴趣。

一个简单的match查询已经通过排序将含有所有搜索词条的文档放在结果列表的前面了。而我们只想对这些前面的结果进行重新排序来给予那些同时匹配了短语查询的文档额外的相关度。

search API通过分值重计算(Rescoring)来支持这一行为。在分值重计算阶段，你能够使用一个更加昂贵的分值计算算法 - 比如一个短语查询 - 来为每个分片的前K个结果重新计算其分值。紧接着这些结果就会按其新的分值重新排序。

该请求如下所示：

```json
GET /my_index/my_type/_search
{
    "query": {
        "match": {  
            "title": {
                "query":                "quick brown fox",
                "minimum_should_match": "30%"
            }
        }
    },
    "rescore": {
        "window_size": 50, 
        "query": {         
            "rescore_query": {
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

match查询用来决定哪些文档会被包含在最终的结果集合中，结果通过TF/IDF进行排序。 window_size是每个分片上需要重新计算分值的数量。


## 寻找关联的单词(Finding Associated Words)

尽管短语和邻近度查询很管用，它们还是有一个缺点。它们过于严格了：所有的在短语查询中的词条都必须出现在文档中，即使使用了slop。

通过slop获得的能够调整单词顺序的灵活性也是有代价的，因为你失去了单词之间的关联。尽管你能够识别文档中的sue，alligator和ate出现在一块，但是你不能判断是Sue ate还是alligator ate。

当单词结合在一起使用时，它们表达的意思比单独使用时要丰富。"I’m not happy I’m working"和"I’m happy I’m not working"含有相同的单词，也拥有相近的邻近度，但是它们的意思大相径庭。

如果我们索引单词对，而不是索引独立的单词，那么我们就能够保留更多关于单词使用的上下文信息。

对于句子"Sue ate the alligator"，我们不仅索引每个单词(或者Unigram)为一个词条：

["sue", "ate", "the", "alligator"]

我们同时会将每个单词和它的邻近单词一起索引成一个词条：

["sue ate", "ate the", "the alligator"]

这些单词对(也叫做Bigram)就是所谓的Shingle。

> TIP
> 
> Shingle不限于只是单词对；你也可以索引三个单词(Word Triplet，也被称为Trigram)作为一个词条：
> 
> ["sue ate the", "ate the alligator"]
> 
> Trigram能够给你更高的精度，但是也大大地增加了索引的不同词条的数量。在多数情况下，Bigram就足够了。

当然，只有当用户输入查询的顺序和原始文档的顺序一致，Shingle才能够起作用；一个针对sue alligator的查询会匹配单独的单词，但是不会匹配任何Shingle。

幸运的是，用户会倾向于使用和他们正在搜索的数据中相似的结构来表达查询。但是这是很重要的一点：仅使用Bigram是不够的；我们仍然需要Unigram，我们可以将匹配Bigram作为信号(Signal)来增加相关度分值。

### 产生Shingle

Shingle需要在索引期间，作为分析过程的一部分被创建。我们可以将Unigram和Bigram都索引到一个字段中，但是将它们放在不同的字段中会更加清晰，也能够让它们能够被独立地查询。Unigram字段形成了我们搜索的基础部分，而Bigram字段则用来提升相关度。

首先，我们需要使用shingle词条过滤器来创建解析器：

```json
DELETE /my_index

PUT /my_index
{
    "settings": {
        "number_of_shards": 1,  
        "analysis": {
            "filter": {
                "my_shingle_filter": {
                    "type":             "shingle",
                    "min_shingle_size": 2, 
                    "max_shingle_size": 2, 
                    "output_unigrams":  false   
                }
            },
            "analyzer": {
                "my_shingle_analyzer": {
                    "type":             "custom",
                    "tokenizer":        "standard",
                    "filter": [
                        "lowercase",
                        "my_shingle_filter" 
                    ]
                }
            }
        }
    }
}
```

默认Shingle的min/max值就是2，因此我们也可以不显式地指定它们。 output_unigrams被设置为false，用来避免将Unigram和Bigram索引到相同字段中。

让我们使用analyze API来测试该解析器：

```
GET /my_index/_analyze?analyzer=my_shingle_analyzer
Sue ate the alligator
```

不出所料，我们得到了3个词条：

- sue ate
- ate the
- the alligator

现在我们就可以创建一个使用新解析器的字段了。

### 多字段(Multifields)

将Unigram和Bigram分开索引会更加清晰，因此我们将title字段创建成一个多字段(Multifield)(参见[字符串排序和多字段(String Sorting and Multifields)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/multi-fields.html))：

```json
PUT /my_index/_mapping/my_type
{
    "my_type": {
        "properties": {
            "title": {
                "type": "string",
                "fields": {
                    "shingles": {
                        "type":     "string",
                        "analyzer": "my_shingle_analyzer"
                    }
                }
            }
        }
    }
}
```

有了上述映射，JSON文档中的title字段会以Unigram(title字段)和Bigram(title.shingles字段)的方式索引，从而让我们可以独立地对这两个字段进行查询。

最后，我们可以索引示例文档：

```json
POST /my_index/my_type/_bulk
{ "index": { "_id": 1 }}
{ "title": "Sue ate the alligator" }
{ "index": { "_id": 2 }}
{ "title": "The alligator ate Sue" }
{ "index": { "_id": 3 }}
{ "title": "Sue never goes anywhere without her alligator skin purse" }
```

### 搜索Shingles

为了理解添加的shingles字段的好处，让我们首先看看一个针对"The hungry alligator ate Sue"的简单match查询的返回结果：

```json
GET /my_index/my_type/_search
{
   "query": {
        "match": {
           "title": "the hungry alligator ate sue"
        }
   }
}
```

该查询会返回所有的3份文档，但是注意文档1和文档2拥有相同的相关度分值，因为它们含有相同的单词：

```json
{
  "hits": [
     {
        "_id": "1",
        "_score": 0.44273707, 
        "_source": {
           "title": "Sue ate the alligator"
        }
     },
     {
        "_id": "2",
        "_score": 0.44273707, 
        "_source": {
           "title": "The alligator ate Sue"
        }
     },
     {
        "_id": "3", 
        "_score": 0.046571054,
        "_source": {
           "title": "Sue never goes anywhere without her alligator skin purse"
        }
     }
  ]
}
```

现在让我们将shingles字段也添加到查询中。记住我们会将shingle字段作为信号 - 以增加相关度分值 - 我们仍然需要将主要的title字段包含到查询中：

```json
GET /my_index/my_type/_search
{
   "query": {
      "bool": {
         "must": {
            "match": {
               "title": "the hungry alligator ate sue"
            }
         },
         "should": {
            "match": {
               "title.shingles": "the hungry alligator ate sue"
            }
         }
      }
   }
}
```

我们仍然匹配了3分文档，但是文档2现在排在了第一位，因为它匹配了Shingle词条"ate sue"：

```json
{
  "hits": [
     {
        "_id": "2",
        "_score": 0.4883322,
        "_source": {
           "title": "The alligator ate Sue"
        }
     },
     {
        "_id": "1",
        "_score": 0.13422975,
        "_source": {
           "title": "Sue ate the alligator"
        }
     },
     {
        "_id": "3",
        "_score": 0.014119488,
        "_source": {
           "title": "Sue never goes anywhere without her alligator skin purse"
        }
     }
  ]
}
```

即使在查询中包含了没有在任何文档中出现的单词hungry，我们仍然通过使用单词邻近度得到了最相关的文档。

### 性能

Shingle不仅比短语查询更灵活，它们的性能也更好。相比每次搜索需要为短语查询付出的代价，对Shingle的查询和简单match查询一样的高效。只是在索引期间会付出一点小代价，因为更多的词条需要被索引，意味着使用了Shingle的字段也会占用更多的磁盘空间。但是，多数应用是写入一次读取多次的，因此在索引期间花费一点代价来让查询更迅速是有意义的。

这是一个你在ES中经常会碰到的主题：让你在搜索期间能够做很多事情，而不需要任何预先的设置。一旦更好地了解了你的需求，就能够通过在索引期间正确地建模来得到更好的结果和更好的性能。


