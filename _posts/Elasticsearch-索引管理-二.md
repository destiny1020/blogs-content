---
title: '[Elasticsearch] 索引管理 (二)'
date: 2014-11-25 10:52:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 索引, Lucene]
---

## 自定义解析器(Custom Analyzers)

虽然ES本身已经提供了一些解析器，但是通过组合字符过滤器(Character Filter)，分词器(Tokenizer)以及词条过滤器(Token Filter)来创建你自己的解析器才会显示出其威力。

在[解析和解析器](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/analysis-intro.html)中，我们提到过解析器(Analyzer)就是将3种功能打包得到的，它会按照下面的顺序执行：

- 字符过滤器(Character Filter) 字符过滤器用来在分词前将字符串进行"整理"。比如，如果文本是HTML格式，那么它会含有类似<p>或者<div>这样的HTML标签，但是这些标签我们是不需要索引的。我们可以使用[html_strip字符过滤器](http://www.elasticsearch.org/guide/en/elasticsearch/reference/1.4//analysis-htmlstrip-charfilter.html)移除所有的HTML标签，并将所有的像Á这样的HTML实体(HTML Entity)转换为对应的Unicode字符：Á。

<!-- More -->

- 分词器(Tokenizers) 一个解析器必须有一个分词器。分词器将字符串分解成一个个单独的词条(Term or Token)。在standard解析器中使用的[standard分词器](http://www.elasticsearch.org/guide/en/elasticsearch/reference/1.4//analysis-standard-tokenizer.html)，通过单词边界对字符串进行划分来得到词条，同时会移除大部分的标点符号。另外还有其他的分词器拥有着不同的行为。

比如[keyword分词器](http://www.elasticsearch.org/guide/en/elasticsearch/reference/1.4//analysis-keyword-tokenizer.html)，它不会进行任何分词，直接原样输出。[whitespace分词器](http://www.elasticsearch.org/guide/en/elasticsearch/reference/1.4//analysis-whitespace-tokenizer.html)则只通过对空白字符进行划分来得到词条。而[pattern分词器](http://www.elasticsearch.org/guide/en/elasticsearch/reference/1.4//analysis-pattern-tokenizer.html)则根据正则表达式来进行分词。

- 词条过滤器(Token Filter) 在分词后，得到的词条流(Token Stream)会按照顺序被传入到指定的词条过滤器中。

词条过滤器能够修改，增加或者删除词条。我们已经提到了[lowercase词条过滤器](http://www.elasticsearch.org/guide/en/elasticsearch/reference/1.4//analysis-lowercase-tokenfilter.html)和[stop词条过滤器](http://www.elasticsearch.org/guide/en/elasticsearch/reference/1.4//analysis-stop-tokenfilter.html)，但是ES中还有许多其它可用的词条过滤器。[stemming词条过滤器](http://www.elasticsearch.org/guide/en/elasticsearch/reference/1.4//analysis-stemmer-tokenfilter.html)会对单词进行词干提取来得到其词根形态(Root Form)。[ascii_folding词条过滤器](http://www.elasticsearch.org/guide/en/elasticsearch/reference/1.4//analysis-asciifolding-tokenfilter.html)则会移除变音符号(Diacritics)，将类似于très的词条转换成tres。[ngram词条过滤器](http://www.elasticsearch.org/guide/en/elasticsearch/reference/1.4//analysis-ngram-tokenfilter.html)和[edge_ngram词条过滤器](http://www.elasticsearch.org/guide/en/elasticsearch/reference/1.4//analysis-edgengram-tokenfilter.html)会产生适用于部分匹配(Partial Matching)或者自动完成(Autocomplete)的词条。

在[深入搜索](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/search-in-depth.html)中，我们会通过例子来讨论这些分词器和过滤器的使用场景和使用方法。但是首先，我们需要解释如何来创建一个自定义的解析器。

### 创建一个自定义的解析器

和上面我们配置es_std解析器的方式相同，我们可以在analysis下对字符过滤器，分词器和词条过滤器进行配置：

```json
PUT /my_index
{
    "settings": {
        "analysis": {
            "char_filter": { ... custom character filters ... },
            "tokenizer":   { ...    custom tokenizers     ... },
            "filter":      { ...   custom token filters   ... },
            "analyzer":    { ...    custom analyzers      ... }
        }
    }
}
```

比如，要创建拥有如下功能的解析器：

1. 使用html_strip字符过滤器完成HTML标签的移除。
2. 将&字符替换成" and "，使用一个自定义的mapping字符过滤器。

```json
"char_filter": {
    "&_to_and": {
        "type":       "mapping",
        "mappings": [ "&=> and "]
    }
}
```

1. 使用standard分词器对文本进行分词。
2. 使用lowercase词条过滤器将所有词条转换为小写。
3. 使用一个自定义的stopword列表，并通过自定义的stop词条过滤器将它们移除：

```json
"filter": {
    "my_stopwords": {
        "type":        "stop",
        "stopwords": [ "the", "a" ]
    }
}
```

我们的解析器将预先定义的分词器和过滤器和自定义的过滤器进行了结合：

```json
"analyzer": {
    "my_analyzer": {
        "type":           "custom",
        "char_filter":  [ "html_strip", "&_to_and" ],
        "tokenizer":      "standard",
        "filter":       [ "lowercase", "my_stopwords" ]
    }
}
```

因此，整个create-index请求就像下面这样：

```json
PUT /my_index
{
    "settings": {
        "analysis": {
            "char_filter": {
                "&_to_and": {
                    "type":       "mapping",
                    "mappings": [ "&=> and "]
            }},
            "filter": {
                "my_stopwords": {
                    "type":       "stop",
                    "stopwords": [ "the", "a" ]
            }},
            "analyzer": {
                "my_analyzer": {
                    "type":         "custom",
                    "char_filter":  [ "html_strip", "&_to_and" ],
                    "tokenizer":    "standard",
                    "filter":       [ "lowercase", "my_stopwords" ]
            }}
}}}
```

创建索引之后，使用analyze API对新的解析器进行测试：

```
GET /my_index/_analyze?analyzer=my_analyzer
The quick & brown fox
```

得到的部分结果如下，表明我们的解析器能够正常工作：

```json
{
  "tokens" : [
      { "token" :   "quick",    "position" : 2 },
      { "token" :   "and",      "position" : 3 },
      { "token" :   "brown",    "position" : 4 },
      { "token" :   "fox",      "position" : 5 }
    ]
}
```

我们需要告诉ES这个解析器应该在什么地方使用。我们可以将它应用在string字段的映射中：

```json
PUT /my_index/_mapping/my_type
{
    "properties": {
        "title": {
            "type":      "string",
            "analyzer":  "my_analyzer"
        }
    }
}
```

## 类型和映射(Types and Mappings)

在ES中的类型(Type)代表的是一类相似的文档。一个类型包含了一个名字(Name) - 比如user或者blogpost - 以及一个映射(Mapping)。映射就像数据库的模式那样，描述了文档中的字段或者属性，和每个字段的数据类型 -string，integer，date等 - 这些字段是如何被Lucene索引和存储的。

在[什么是文档](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/document.html)中，我们说一个类型就好比关系数据库中的一张表。尽管一开始这样思考有助于理解，但是对类型本身进行更细致的解释 - 它们到底是什么，它们是如何在Lucene的基础之上实现的 - 仍然是有价值的。

### Lucene是如何看待文档的

Lucene中的文档包含的是一个简单field-value对的列表。一个字段至少要有一个值，但是任何字段都可以拥有多个值。类似的，一个字符串值也可以通过解析阶段而被转换为多个值。Lucene不管值是字符串类型，还是数值类型或者什么别的类型 - 所有的值都会被同等看做一些不透明的字节(Opaque bytes)。

当我们使用Lucene对文档进行索引时，每个字段的值都会被添加到倒排索引(Inverted Index)的对应字段中。原始值也可以被选择是否会不作修改的被保存到索引中，以此来方便将来的获取。

### 类型是如何实现的

ES中的type是基于以下简单的基础进行实现的。一个索引中可以有若干个类型，每个类型又有它自己的mapping，然后类型下的任何文档可以存储在同一个索引中。

可是Lucene中并没有文档类型这一概念。所以在具体实现中，类型信息通过一个元数据字段_type记录在文档中。当我们需要搜索某个特定类型的文档时，ES会自动地加上一个针对_type字段的过滤器来保证返回的结果都是目标类型上的文档。

同时，Lucene中也没有映射的概念。映射是ES为了对复杂JSON文档进行扁平化(可以被Lucene索引)而设计的一个中间层。

比如，user类型的name字段可以定义成一个string类型的字段，而它的值则应该被whitespace解析器进行解析，然后再被索引到名为name的倒排索引中。

```json
"name": {
    "type":     "string",
    "analyzer": "whitespace"
}
```

### 避免类型中的陷阱

由于不同类型的文档能够被添加到相同的索引中，产生了一些意想不到的问题。

比如在我们的索引中，存在两个类型：blog_en用来保存英文的博文，blog_es用来保存西班牙文的博文。这两种类型中都有一个title字段，只不过它们使用的解析器分别是english和spanish。

问题可以通过下面的查询反映：

```json
GET /_search
{
    "query": {
        "match": {
            "title": "The quick brown fox"
        }
    }
}
```

我们在两个类型中搜索title字段。查询字符串(Query String)需要被解析，但是应该使用哪个解析器：是spanish还是english？答案是会利用首先找到的title字段对应的解析器，因此对于部分文档这样做是正确的，对于另一部分则不然。

我们可以通过将字段命名地不同 - 比如title_en和title_es - 或者通过显式地将类型名包含在字段名中，然后对每个字段独立查询来避免这个问题：

```json
GET /_search
{
    "query": {
        "multi_match": { 
            "query":    "The quick brown fox",
            "fields": [ "blog_en.title", "blog_es.title" ]
        }
    }
}
```

multi_match查询会对指定的多个字段运行match查询，然后合并它们的结果。

以上的查询中对blog_en.title字段使用english解析器，对blog_es.title字段使用spanish解析器，然后对两个字段的搜索结果按照相关度分值进行合并。

这个解决方案能够在两个域是相同数据类型时起作用，但是考虑下面的场景，当向相同索引中添加两份文档时会发生什么：

**类型user**

```json
{ "login": "john_smith" }
```

**类型event**

```json
{ "login": "2014-06-01" }
```

Lucene本身不在意类型一个字段是字符串类型，而另一个字段是日期类型 - 它只是愉快地将它们当做字节数据进行索引。

但是当我们试图去针对event.login字段进行排序的时候，ES需要将login字段的值读入到内存中。根据[Fielddata](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/fielddata-intro.html)提到的，ES会将索引中的所有文档都读入，无论其类型是什么。

取决于ES首先发现的login字段的类型，它会试图将这些值当做字符串或者日期类型读入。因此，这会产生意料外的结果或者直接失败。

> Tip 
> 
> 为了避免发生这些冲突，建议索引中，每个类型的同名字段都使用相同的映射方式。