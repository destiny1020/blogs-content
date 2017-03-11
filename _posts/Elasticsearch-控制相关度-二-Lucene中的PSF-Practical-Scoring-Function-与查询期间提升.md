---
title: '[Elasticsearch] 控制相关度 (二) - Lucene中的PSF(Practical Scoring Function)与查询期间提升'
date: 2014-12-24 10:12:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 相关度分值计算]
---

本章翻译自Elasticsearch官方指南的[Controlling Relevance](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/controlling-relevance.html)一章。

## Lucene中的Practical Scoring Function

对于多词条查询(Multiterm Queries)，Lucene使用的是[布尔模型(Boolean Model)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/scoring-theory.html#boolean-model)，[TF/IDF](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/scoring-theory.html#tfidf)以及[向量空间模型(Vector Space Model)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/scoring-theory.html#vector-space-model)来将它们结合在一起，用来收集匹配的文档和对它们进行分值计算。

<!-- More -->

像下面这样的多词条查询：

```json
GET /my_index/doc/_search
{
  "query": {
    "match": {
      "text": "quick fox"
    }
  }
}
```

在内部被重写成下面这样：

```json
GET /my_index/doc/_search
{
  "query": {
    "bool": {
      "should": [
        {"term": { "text": "quick" }},
        {"term": { "text": "fox"   }}
      ]
    }
  }
}
```

bool查询实现了布尔模型，在这个例子中，只有包含了词条quick，词条fox或者两者都包含的文档才会被返回。

一旦一份文档匹配了一个查询，Lucene就会为该查询计算它的分值，然后将每个匹配词条的分值结合起来。用来计算分值的公式叫做Practical Scoring Function。它看起来有点吓人，但是不要退却 - 公式中的绝大多数部分你已经知道了。下面我们会介绍它引入的一些新元素。

```
1   score(q,d)  = 
2            queryNorm(q)  
3          · coord(q,d)    
4          · ∑ (           
5                tf(t in d)   
6              · idf(t)²      
7              · t.getBoost() 
8              · norm(t,d)    
9            ) (t in q) 
```

每行的意义如下：

1. score(q,d)是文档d对于查询q的相关度分值。
2. queryNorm(q)是[查询归约因子(Query Normalization Factor)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/practical-scoring-function.html#query-norm)，是新添加的部分。
3. coord(q,d)是[Coordination Factor](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/practical-scoring-function.html#coord)，是新添加的部分。
4. 文档d中每个词条t对于查询q的权重之和。
5. tf(t in d)是文档d中的词条t的[词条频度(Term Frequency)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/scoring-theory.html#tf)。
6. idf(t)是词条t的[倒排索引频度(Inverse Document Frequency)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/scoring-theory.html#idf)
7. t.getBoost()是适用于查询的[提升(Boost)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/query-time-boosting.html)，是新添加的部分。
8. norm(t,d)是[字段长度归约(Field-length Norm)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/scoring-theory.html#field-norm)，可能结合了[索引期间字段提升(Index-time Field-level Boost)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/practical-scoring-function.html#index-boost)，是新添加的部分。
你应该知道score，tf以及idf的意思。queryNorm，coord，t.getBoost以及norm是新添加的。

在本章的稍后我们会讨论[查询期间提升(Query-time Boosting)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/query-time-boosting.html)，首先对查询归约，Coordination以及索引期间字段级别提升进行解释。

### 查询归约因子(Query Normalization Factor)

查询归约因子(queryNorm)会试图去对一个查询进行归约，从而让多个查询的结果能够进行比较。

> TIP
> 
> 虽然查询归约的目的是让不同查询的结果能够比较，但是它的效果不怎么好。相关度_score的唯一目的是将当前查询的结果以正确的顺序被排序。你不应该尝试去比较不同查询得到的相关度分值。

该因子会在查询开始阶段就被计算。实际的计算取决于查询本身，但是一个典型的实现如下所示：

> queryNorm = 1 / √sumOfSquaredWeights

sumOfSquaredWeights通过对查询中每个词条的IDF进行累加，然后取其平方根得到的。
> 
> TIP
> 
> 相同的查询归约因子会被适用在每份文档上，你也没有办法改变它。总而言之，它是可以被忽略的。

### Query Coordination

Coordination因子(coord)被用来奖励那些包含了更多查询词条的文档。文档中出现了越多的查询词条，那么该文档就越可能是该查询的一个高质量匹配。

加入我们查询了quick brown fox，每个词条的权重都是1.5。没有Coordination因子时，分值可能会是文档中每个词条的权重之和。比如：

- 含有fox的文档 -> 分值：1.5
- 含有quick fox的文档 -> 分值：3.0
- 含有quick brown fox的文档 -> 分值：4.5

而Coordination因子会将分值乘以文档中匹配了的词条的数量，然后除以查询中的总词条数。使用了Coordination因子后，分值是这样的：

- 含有fox的文档 -> 分值：1.5 * 1 / 3 = 0.5
- 含有quick fox的文档 -> 分值：3.0 * 2 / 3 = 2.0
- 含有quick brown fox的文档 -> 分值：4.5 * 3 / 3 = 4.5

以上的结果中，含有所有三个词条的文档的分值会比仅含有两个词条的文档高出许多。

需要记住对于quick brown fox的查询会被bool查询重写如下：

```json
GET /_search
{
  "query": {
    "bool": {
      "should": [
        { "term": { "text": "quick" }},
        { "term": { "text": "brown" }},
        { "term": { "text": "fox"   }}
      ]
    }
  }
}
```

bool查询会对所有should查询子句默认启用查询Coordination，但是你可以禁用它。为什么你需要禁用它呢？好吧，通常的答案是，并不需要。查询Coordination通常都起了正面作用。当你使用bool查询来将多个像match这样的高级查询(High-level Query)包装在一起时，启用Coordination也是有意义的。匹配的查询子句越多，你的搜索陈请求和返回的文档之间的匹配程度就越高。

但是，在某些高级用例中，禁用Coordination也是有其意义的。比如你正在查询同义词jump，leap和hop。你不需要在意这些同义词出现了多少次，因为它们表达了相同的概念。实际上，只有其中的一个可能会出现。此时，禁用Coordination因子就是一个不错的选择：

```json
GET /_search
{
  "query": {
    "bool": {
      "disable_coord": true,
      "should": [
        { "term": { "text": "jump" }},
        { "term": { "text": "hop"  }},
        { "term": { "text": "leap" }}
      ]
    }
  }
}
```

当你使用了同义词(参考[同义词(Synonyms)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/synonyms.html))，这正是在内部发生的：重写的查询会为同义词禁用Coordination。多数禁用Coordination的用例都会被自动地处理；你根本无需担心它。

### 索引期间字段级别提升(Index-time Field-level Boosting)

现在来讨论一下字段提升 - 让该字段比其它字段更重要一些 - 通过在查询期间使用[查询期间提升(Query-time Boosting)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/query-time-boosting.html)。在索引期间对某个字段进行提升也是可能的。实际上，该提升会适用于字段的每个词条上，而不是在字段本身。

为了在尽可能少占用空间的前提下，将提升值存储到索引中，索引期间字段级别提升会和[字段长度归约](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/scoring-theory.html#field-norm)一起以一个字节被保存在索引中。它是之前公式中norm(t,d)返回的值。

> 警告
> 
> 我们强烈建议不要使用字段级别索引期间提升的原因如下：
> 
> - 将此提升和字段长度归约存储在一个字节中意味着字段长度归约会损失精度。结果是ES不能区分一个含有三个单词的字段和一个含有五个单词的字段。
> - 为了修改索引期间提升，你不得不对所有文档重索引。而查询期间的提升则可以因查询而异。
> - 如果一个使用了索引期间提升的字段是多值字段(Multivalue Field)，那么提升值会为每一个值进行乘法操作，导致该字段的权重飙升。
> 
> [查询期间提升(Query-time Boosting)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/query-time-boosting.html)更简单，简洁和灵活。

解释完了查询归约，Coordination以及索引期间提升，现在可以开始讨论对影响相关度计算最有用的工具：查询期间提升。

## 查询期间提升(Query-time Boosting)

在[调整查询子句优先级(Prioritizing Clauses)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/multi-query-strings.html#prioritising-clauses)一节中，我们已经介绍过如何在搜索期间使用boost参数为一个查询子句增加权重。比如：

```json
GET /_search
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
            "title": {
              "query": "quick brown fox",
              "boost": 2 
            }
          }
        },
        {
          "match": { 
            "content": "quick brown fox"
          }
        }
      ]
    }
  }
}
```

查询期间提升是用来调优相关度的主要工具。任何类型的查询都接受boost参数。将boost设为2并不是简单地将最终的_score加倍；确切的提升值会经过规范化以及一些内部优化得到。但是，它也意味着一个提升值为2的子句比一个提升值为1的子句要重要两倍。

实际上，没有任何公式能够决定对某个特定的查询子句，"正确的"提升值应该是多少。它是通过尝试来得到的。记住boost仅仅是相关度分值中的一个因素；它需要和其它因素竞争。比如在上面的例子中，title字段相对于content字段，大概已经有一个"自然的"提升了，该提升来自[字段长度归约(Field-length Norm)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/scoring-theory.html#field-norm)(因为标题通常会比相关内容要短一些)，因此不要因为你认为某个字段应该被提升而盲目地对它进行提升。适用一个提升值然后检查得到的结果，再进行修正。

### 提升索引(Boosting an Index)

当在多个索引中搜索时，你可以通过indices_boost参数对整个索引进行提升。在下面的例子中，会给予最近索引中的文档更多的权重：

```json
GET /docs_2014_*/_search 
{
  "indices_boost": { 
    "docs_2014_10": 3,
    "docs_2014_09": 2
  },
  "query": {
    "match": {
      "text": "quick brown fox"
    }
  }
}
```

该多索引搜索(Multi-index Search)会查询所有以docs_2014_开头的索引。 索引docs_2014_10中的文档的提升值为3，索引docs_2014_09中的文档的提升值为2，其它索引中的文档的提升值为默认值1。

### t.getBoost()

这些提升值在[Lucene的Practical Scoring Function](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/practical-scoring-function.html)中通过t.getBoost()元素表达。提升并不是其在查询DSL出现的地方被适用的。相反，任何的提升值都会被合并然后传递到每个词条上。t.getBoost()方法返回的是适用于词条本身上的提升值，或者是适用于上层查询的提升值。

> TIP
> 
> 实际上，阅读[解释API](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/relevance-intro.html#explain)的输出本身比上述的说明更复杂。你在解释中根本看不到boost值或者t.getBoost()。提升被融合到了适用于特定词条上的[queryNorm](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/practical-scoring-function.html#query-norm)中。尽管我们说过queryNorm对任何词条都是相同的，但是对于提升过的词条而言，queryNorm会更高一些。