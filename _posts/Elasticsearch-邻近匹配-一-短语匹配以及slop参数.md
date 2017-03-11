---
title: '[Elasticsearch] 邻近匹配 (一) - 短语匹配以及slop参数'
date: 2014-12-15 11:31:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 全文搜索]
---

本文翻译自Elasticsearch官方指南的[Proximity Matching](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/partial-matching.html)一章。

# 邻近匹配(Proximity Matching)

使用了TF/IDF的标准全文搜索将文档，或者至少文档中的每个字段，视作"一大袋的单词"(Big bag of Words)。match查询能够告诉我们这个袋子中是否包含了我们的搜索词条，但是这只是一个方面。它不能告诉我们关于单词间关系的任何信息。

考虑以下这些句子的区别：

- Sue ate the alligator.
- The alligator ate Sue.
- Sue never goes anywhere without her alligator-skin purse.

<!-- More -->

一个使用了sue alligator的match查询会匹配以上所有文档，但是它无法告诉我们这两个词是否表达了部分原文的部分意义，或者是表达了完整的意义。

理解单词间的联系是一个复杂的问题，我们也无法仅仅依靠另一类查询就解决这个问题，但是我们至少可以通过单词间的距离来判断单词间可能的关系。

真实的文档也许比上面几个例子要长的多：Sue和alligator也许相隔了几个段落。也许我们仍然希望包含这样的文档，但是我们会给那些Sue和alligator出现的较近的文档更高的相关度分值。

这就是短语匹配(Phrase Matching)，或者邻近度匹配(Proximity Matching)。

> TIP
> 
> 本章中，我们仍然会使用[match查询](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/match-query.html#match-test-data)中使用的示例文档。

## 短语匹配(Phrase Matching)

就像一提到全文搜索会首先想到match查询一样，当你需要寻找邻近的几个单词时，你会使用match_phrase查询：

```json
GET /my_index/my_type/_search
{
    "query": {
        "match_phrase": {
            "title": "quick brown fox"
        }
    }
}
```

和match查询类似，match_phrase查询首先解析查询字符串来产生一个词条列表。然后会搜索所有的词条，但只保留含有了所有搜索词条的文档，并且词条的位置要邻接。一个针对短语quick fox的查询不会匹配我们的任何文档，因为没有文档含有邻接在一起的quick和box词条。

> TIP
> 
> match_phrase查询也可以写成类型为phrase的match查询：
> 
> ```json
> "match": {
>     "title": {
>         "query": "quick brown fox",
>         "type":  "phrase"
>     }
> }
> ```

### 词条位置

当一个字符串被解析时，解析器不仅只返回一个词条列表，它同时也返回每个词条的位置，或者顺序信息：

```
GET /_analyze?analyzer=standard
Quick brown fox
```

会返回以下的结果：

```json
{
   "tokens": [
      {
         "token": "quick",
         "start_offset": 0,
         "end_offset": 5,
         "type": "<ALPHANUM>",
         "position": 1 
      },
      {
         "token": "brown",
         "start_offset": 6,
         "end_offset": 11,
         "type": "<ALPHANUM>",
         "position": 2 
      },
      {
         "token": "fox",
         "start_offset": 12,
         "end_offset": 15,
         "type": "<ALPHANUM>",
         "position": 3 
      }
   ]
}
```

位置信息可以被保存在倒排索引(Inverted Index)中，像match_phrase这样位置感知(Position-aware)的查询能够使用位置信息来匹配那些含有正确单词出现顺序的文档，在这些单词间没有插入别的单词。

### 短语是什么

对于匹配了短语"quick brown fox"的文档，下面的条件必须为true：

- quick，brown和fox必须全部出现在某个字段中。
- brown的位置必须比quick的位置大1。
- fox的位置必须比quick的位置大2。

如果以上的任何条件没有被满足，那么文档就不能被匹配。

> TIP
> 
> 在内部，match_phrase查询使用了低级的span查询族(Query Family)来执行位置感知的查询。span查询是词条级别的查询，因此它们没有解析阶段(Analysis Phase)；它们直接搜索精确的词条。
> 
> 幸运的是，大多数用户几乎不需要直接使用span查询，因为match_phrase查询通常已经够好了。但是，对于某些特别的字段，比如专利搜索(Patent Search)，会使用这些低级查询来执行拥有非常特别构造的位置搜索。

## 混合起来(Mixing it up)

精确短语(Exact-phrase)匹配也许太过于严格了。也许我们希望含有"quick brown fox"的文档也能够匹配"quick fox"查询，即使位置并不是完全相等的。

我们可以在短语匹配使用slop参数来引入一些灵活性：

```json
GET /my_index/my_type/_search
{
    "query": {
        "match_phrase": {
            "title": {
                "query": "quick fox",
                "slop":  1
            }
        }
    }
}
```

slop参数告诉match_phrase查询词条能够相隔多远时仍然将文档视为匹配。相隔多远的意思是，你需要移动一个词条多少次来让查询和文档匹配？

我们以一个简单的例子来阐述这个概念。为了让查询quick fox能够匹配含有quick brown fox的文档，我们需要slop的值为1：

```
            Pos 1         Pos 2         Pos 3
-----------------------------------------------
Doc:        quick         brown         fox
-----------------------------------------------
Query:      quick         fox
Slop 1:     quick                 ↳     fox
```

尽管在使用了slop的短语匹配中，所有的单词都需要出现，但是单词的出现顺序可以不同。如果slop的值足够大，那么单词的顺序可以是任意的。

为了让fox quick查询能够匹配我们的文档，需要slop的值为3：

```
            Pos 1         Pos 2         Pos 3
-----------------------------------------------
Doc:        quick         brown         fox
-----------------------------------------------
Query:      fox           quick
Slop 1:     fox|quick  ↵  
Slop 2:     quick      ↳  fox
Slop 3:     quick                 ↳     fox
```