---
title: '[Elasticsearch] 全文搜索 (一) - 基础概念和match查询'
date: 2014-12-03 10:04:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 全文搜索]
---

翻译自官方指南的[全文搜索](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/full-text-search.html)一章。

## 全文搜索(Full Text Search)

现在我们已经讨论了搜索结构化数据的一些简单用例，是时候开始探索全文搜索了 - 如何在全文字段中搜索来找到最相关的文档。

对于全文搜索而言，最重要的两个方面是：

<!-- More -->

**相关度(Relevance)**

查询的结果按照它们对查询本身的相关度进行排序的能力，相关度可以通过TF/IDF，参见[什么是相关度](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/relevance-intro.html)，地理位置的邻近程度(Proximity to a Geo-location)，模糊相似性(Fuzzy Similarity)或者其它算法进行计算。

**解析(Analysis)**

解析用来将一块文本转换成单独的，规范化的词条(Tokens)，参见[解析和解析器(Analysis and Analyzers)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/analysis-intro.html)，用来完成：(a)倒排索引(Inverted Index)的创建；(b)倒排索引的查询。

一旦我们开始讨论相关度或者解析，也就意味着我们踏入了查询(Query)的领域，而不再是过滤器(Filter)。

### 基于词条(Term-based)和全文(Full-text)

尽管所有的查询都会执行某种程度的相关度计算，并不是所有的查询都存在解析阶段。除了诸如bool或者function_score这类完全不对文本进行操作的特殊查询外，对于文本的查询可以被划分两个种类：

#### 基于词条的查询(Term-based Queries)

类似term和fuzzy的查询是不含有解析阶段的低级查询(Low-level Queries)。它们在单一词条上进行操作。一个针对词条Foo的term查询会在倒排索引中寻找该词条的精确匹配(Exact term)，然后对每一份含有该词条的文档通过TF/IDF进行相关度_score的计算。

尤其需要记住的是term查询只会在倒排索引中寻找该词条的精确匹配 - 它不会匹配诸如foo或者FOO这样的变体。它不在意词条是如何被保存到索引中。如果你索引了["Foo", "Bar"]到一个not_analyzed字段中，或者将Foo Bar索引到一个使用whitespace解析器的解析字段(Analyzed Field)中，它们都会在倒排索引中得到两个词条："Foo"以及"Bar"。

#### 全文查询(Full-text Queries)

类似match或者query_string这样的查询是高级查询(High-level Queries)，它们能够理解一个字段的映射：

- 如果你使用它们去查询一个date或者integer字段，它们会将查询字符串分别当做日期或者整型数。
- 如果你查询一个精确值(not_analyzed)字符串字段，它们会将整个查询字符串当做一个单独的词条。
- 但是如果你查询了一个全文字段(analyzed)，它们会首先将查询字符串传入到合适的解析器，用来得到需要查询的词条列表。
一旦查询得到了一个词条列表，它就会使用列表中的每个词条来执行合适的低级查询，然后将得到的结果进行合并，最终产生每份文档的相关度分值。

我们会在后续章节中详细讨论这个过程。

---

在很少的情况下，你才需要直接使用基于词条的查询(Term-based Queries)。通常你需要查询的是全文，而不是独立的词条，而这个工作通过高级的全文查询来完成会更加容易(在内部它们最终还是使用的基于词条的低级查询)。

如果你发现你确实需要在一个not_analyzed字段上查询一个精确值，那么考虑一下你是否真的需要使用查询，而不是使用过滤器。

单词条查询通常都代表了一个二元的yes|no问题，这类问题通常使用过滤器进行表达更合适，因此它们也能够得益于[过滤器缓存(Filter Caching)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/filter-caching.html)：

```json
GET /_search
{
    "query": {
        "filtered": {
            "filter": {
                "term": { "gender": "female" }
            }
        }
    }
}
```

#### match查询

在你需要对任何字段进行查询时，match查询应该是你的首选。它是一个高级全文查询，意味着它知道如何处理全文字段(Full-text, analyzed)和精确值字段(Exact-value，not_analyzed)。

即便如此，match查询的主要使用场景仍然是全文搜索。让我们通过一个简单的例子来看看全文搜索时如何工作的。

索引一些数据

首先，我们会创建一个新的索引并通过[bulk API](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/bulk.html)索引一些文档：

```json
DELETE /my_index 

PUT /my_index
{ "settings": { "number_of_shards": 1 }} 

POST /my_index/my_type/_bulk
{ "index": { "_id": 1 }}
{ "title": "The quick brown fox" }
{ "index": { "_id": 2 }}
{ "title": "The quick brown fox jumps over the lazy dog" }
{ "index": { "_id": 3 }}
{ "title": "The quick brown fox jumps over the quick dog" }
{ "index": { "_id": 4 }}
{ "title": "Brown fox brown dog" }
```

注意到以上在创建索引时，我们设置了number_of_shards为1：在稍后的[相关度坏掉了(Relevance is broken)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/relevance-is-broken.html)一节中，我们会解释为何这里创建了一个只有一个主分片(Primary shard)的索引。

#### 单词查询(Single word query)

第一个例子我们会解释在使用match查询在一个全文字段中搜索一个单词时，会发生什么：

```json
GET /my_index/my_type/_search
{
    "query": {
        "match": {
            "title": "QUICK!"
        }
    }
}
```

ES会按照如下的方式执行上面的match查询：

1. 检查字段类型

	title字段是一个全文字符串字段(analyzed)，意味着查询字符串也需要被分析。

2. 解析查询字符串

	查询字符串"QUICK!"会被传入到标准解析器中，得到的结果是单一词条"quick"。因为我们得到的只有一个词条，match查询会使用一个term低级查询来执行查询。

3. 找到匹配的文档

	term查询会在倒排索引中查询"quick"，然后获取到含有该词条的文档列表，在这个例子中，文档1，2，3会被返回。

4. 对每份文档打分

	term查询会为每份匹配的文档计算其相关度分值_score，该分值通过综合考虑词条频度(Term Frequency)("quick"在匹配的每份文档的title字段中出现的频繁程度)，倒排频度(Inverted Document Frequency)("quick"在整个索引中的所有文档的title字段中的出现程度)，以及每个字段的长度(较短的字段会被认为相关度更高)来得到。参考[什么是相关度(What is Relevance?)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/relevance-intro.html)

这个过程会给我们下面的结果(有省略)：

```json
"hits": [
 {
    "_id":      "1",
    "_score":   0.5, 
    "_source": {
       "title": "The quick brown fox"
    }
 },
 {
    "_id":      "3",
    "_score":   0.44194174, 
    "_source": {
       "title": "The quick brown fox jumps over the quick dog"
    }
 },
 {
    "_id":      "2",
    "_score":   0.3125, 
    "_source": {
       "title": "The quick brown fox jumps over the lazy dog"
    }
 }
]
```

文档1最相关，因为它的title字段短，意味着quick在它所表达的内容中占比较大。 文档3比文档2的相关度更高，因为quick出现了两次。
