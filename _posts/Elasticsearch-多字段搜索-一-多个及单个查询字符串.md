---
title: '[Elasticsearch] 多字段搜索 (一) - 多个及单个查询字符串'
date: 2014-12-08 10:19:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 全文搜索]
---

# 多字段搜索(Multifield Search)

本文翻译自官方指南的[Multifield Search](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/multi-field-search.html)一章。

查询很少是只拥有一个match查询子句的查询。我们经常需要对一个或者多个字段使用相同或者不同的查询字符串进行搜索，这意味着我们需要将多个查询子句和它们得到的相关度分值以一种有意义的方式进行合并。

也许我们正在寻找一本名为战争与和平的书，它的作者是Leo Tolstoy。也许我们正在使用"最少应该匹配(Minimum Should Match)"来搜索ES中的文档。另外我们也可能会寻找拥有名为John而姓为Smith的用户。

在本章中我们会讨论一些构建多字段搜索的工具，以及如何根据你的实际情况来决定使用哪种方案。

<!-- More -->

## 多个查询字符串(Multiple Query Strings)

处理字段查询最简单的方法是将搜索词条对应到特定的字段上。如果我们知道战争与和平是标题，而Leo Tolstoy是作者，那么我们可以简单地将每个条件当做一个match子句，然后通过[bool查询](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/bool-query.html)将它们合并：

```json
GET /_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title":  "War and Peace" }},
        { "match": { "author": "Leo Tolstoy"   }}
      ]
    }
  }
}
```

bool查询采用了一种"匹配越多越好(More-matches-is-better)"的方法，因此每个match子句的分值会被累加来得到文档最终的_score。匹配两个子句的文档相比那些只匹配一个子句的文档的分值会高一些。

当然，你并不是只能使用match子句：bool查询可以包含任何其他类型的查询，包括其它的bool查询。我们可以添加一个子句来指定我们希望的译者：

```json
GET /_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title":  "War and Peace" }},
        { "match": { "author": "Leo Tolstoy"   }},
        { "bool":  {
          "should": [
            { "match": { "translator": "Constance Garnett" }},
            { "match": { "translator": "Louise Maude"      }}
          ]
        }}
      ]
    }
  }
}
```

我们为什么将译者的查询子句放在一个单独的bool查询中？所有的4个match查询都是should子句，那么为何不将译者的查询子句和标题及作者的查询子句放在同一层次上呢？

答案在于分值是如何计算的。bool查询会运行每个match查询，将它们的分值相加，然后乘以匹配的查询子句的数量，最后除以所有查询子句的数量。相同层次的每个子句都拥有相同的权重。在上述查询中，bool查询中包含的译者查询子句只占了总分值的三分之一。如果我们将译者查询子句放到和标题及作者相同的层次上，就会减少标题和作者子句的权重，让它们各自只占四分之一。

### 设置子句优先级

上述查询中每个子句占有三分之一的权重也许并不是我们需要的。相比译者字段，我们可能对标题和作者字段更有兴趣。我们对查询进行调整来让标题和作者相对更重要。

在所有可用措施中，我们可以采用的最简单的方法是boost参数。为了增加title和author字段的权重，我们可以给它们一个大于1的boost值：

```json
GET /_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { 
            "title":  {
              "query": "War and Peace",
              "boost": 2
        }}},
        { "match": { 
            "author":  {
              "query": "Leo Tolstoy",
              "boost": 2
        }}},
        { "bool":  { 
            "should": [
              { "match": { "translator": "Constance Garnett" }},
              { "match": { "translator": "Louise Maude"      }}
            ]
        }}
      ]
    }
  }
}
```

以上的title和k字段的boost值为2。 嵌套的bool查询自居的默认boost值为k。

通过试错(Trial and Error)的方式可以确定"最佳"的boost值：设置一个boost值，执行测试查询，重复这个过程。一个合理boost值的范围在1和10之间，也可能是15。比它更高的值的影响不会起到很大的作用，因为分值会被[规范化(Normalized)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/_boosting_query_clauses.html#boost-normalization)。

## 单一查询字符串(Single Query String)

bool查询是多字段查询的中流砥柱。在很多场合下它都能很好地工作，特别是当你能够将不同的查询字符串映射到不同的字段时。

问题在于，现在的用户期望能够在一个地方输入所有的搜索词条，然后应用能够知道如何为他们得到正确的结果。所以当我们把含有多个字段的搜索表单称为高级搜索(Advanced Search)时，是有一些讽刺意味的。高级搜索虽然对用户而言会显得更"高级"，但是实际上它的实现方式更简单。

对于多词，多字段查询并没有一种万能的方法。要得到最佳的结果，你需要了解你的数据以及如何使用恰当的工具。

### 了解你的数据

当用户的唯一输入就是一个查询字符串时，你会经常碰到以下三种情况：

**最佳字段(Best fields)**

当搜索代表某些概念的单词时，例如"brown fox"，几个单词合在一起表达出来的意思比单独的单词更多。类似title和body的字段，尽管它们是相关联的，但是也是互相竞争着的。文档在相同的字段中应该有尽可能多的单词(译注：搜索的目标单词)，文档的分数应该来自拥有最佳匹配的字段。

**多数字段(Most fields)**

一个用来调优相关度的常用技术是将相同的数据索引到多个字段中，每个字段拥有自己的分析链(Analysis Chain)。

主要字段会含有单词的词干部分，同义词和消除了变音符号的单词。它用来尽可能多地匹配文档。

相同的文本可以被索引到其它的字段中来提供更加精确的匹配。一个字段或许会包含未被提取词干的单词，另一个字段是包含了变音符号的单词，第三个字段则使用shingle来提供关于[单词邻近度(Word Proximity)](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/proximity-matching.html)的信息。

以上这些额外的字段扮演者signal的角色，用来增加每个匹配的文档的相关度分值。越多的字段被匹配则意味着文档的相关度越高。

**跨字段(Cross fields)**

对于一些实体，标识信息会在多个字段中出现，每个字段中只含有一部分信息：

- Person：first_name 和 last_name
- Book：title，author 和 description
- Address：street，city，country 和 postcode

此时，我们希望在任意字段中找到尽可能多的单词。我们需要在多个字段中进行查询，就好像这些字段是一个字段那样。

以上这些都是多词，多字段查询，但是每种都需要使用不同的策略。我们会在本章剩下的部分解释每种策略。


