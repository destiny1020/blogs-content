---
title: '[Elasticsearch] 索引管理 (四) - 动态映射'
date: 2014-11-29 11:11:00
categories: [译文, Elasticsearch]
tags: [Elasticsearch, 索引]
---

## 动态映射(Dynamic Mapping)

当ES在文档中碰到一个以前没见过的字段时，它会利用动态映射来决定该字段的类型，并自动地对该字段添加映射。

有时这正是需要的行为，但有时不是。你或许不知道在以后你的文档中会添加哪些字段，但是你想要它们能够被自动地索引。或许你只是想要忽略它们。或者 - 尤其当你将ES当做主要的数据存储使用时 - 大概你会希望这些未知的字段会抛出异常来提醒你注意这一问题。

<!-- More -->

幸运的是，你可以通过dynamic设置来控制这一行为，它能够接受以下的选项：

- true：默认值。动态添加字段
- false：忽略新字段
- strict：如果碰到陌生字段，抛出异常

dynamic设置可以适用在根对象上或者object类型的任意字段上。你应该默认地将dynamic设置为strict，但是为某个特定的内部对象启用它：

```json
PUT /my_index
{
    "mappings": {
        "my_type": {
            "dynamic":      "strict", 
            "properties": {
                "title":  { "type": "string"},
                "stash":  {
                    "type":     "object",
                    "dynamic":  true 
                }
            }
        }
    }
}
```

在my_type对象上如果碰到了未知字段则会抛出一个异常。 在stash对象上会动态添加新字段。

通过以上的映射，你可以向stash添加新的可搜索的字段：

```json
PUT /my_index/my_type/1
{
  "title": "This doc adds a new field",
  "stash": {
    "new_field": "Success!"
  }
}
```

但是，如果在顶层对象上试图添加新字段则会失败：

```json
PUT /my_index/my_type/1
{
    "title":     "This throws a StrictDynamicMappingException",
    "new_field": "Fail!"
}
```

> NOTE
> 
> 将dynamic设置为false并不会改变_source字段的内容 - _source字段仍然会保存你索引的整个JSON文档。只不过是陌生的字段将不会被添加到映射中，以至于它不能被搜索到。

## 自定义动态映射

如果你知道你需要动态的添加的新字段，那么你也许会启用动态映射。然而有时动态映射的规则又有些不够灵活。幸运的是，你可以调整某些设置来让动态映射的规则更加适合你的数据。

### date_detection

当ES碰到一个新的字符串字段时，它会检查该字串是否含有一个可被识别的日期，比如2014-01-01。如果存在，那么它会被识别为一个date类型的字段。否则会将它作为string进行添加。

有时这种行为会导致一些问题。如果你想要索引一份这样的文档：

```json
{ "note": "2014-01-01" }
```

假设这是note字段第一次被发现，那么根据规则它会被作为date字段添加。但是如果下一份文档是这样的：

```json
{ "note": "Logged out" }
```

这时该字段显然不是日期，但是已经太迟了。该字段的类型已经是日期类型的字段了，因此这会导致一个异常被抛出。

可以通过在根对象上将date_detection设置为false来关闭日期检测：

```json
PUT /my_index
{
    "mappings": {
        "my_type": {
            "date_detection": false
        }
    }
}
```

有了以上的映射，一个字符串总是会被当做string类型。如果你需要一个date字段，你需要手动地添加它。

> NOTE
> 
> ES中识别日期的方法可以通过[dynamic_date_formats设置](http://www.elasticsearch.org/guide/en/elasticsearch/reference/1.4//mapping-root-object-type.html#_dynamic_date_formats)改变。

### dynamic_templates

通过dynamic_templates，你可以拥有对新字段的动态映射规则拥有完全的控制。你设置可以根据字段名称或者类型来使用一个不同的映射规则。

每个模板都有一个名字，可以用来描述这个模板做了什么。同时它有一个mapping用来指定具体的映射信息，和至少一个参数(比如match)用来规定对于什么字段需要使用该模板。

模板的匹配是有顺序的 - 第一个匹配的模板会被使用。比如我们可以为string字段指定两个模板：

- es：以_es结尾的字段应该使用spanish解析器
- en：其它所有字段使用english解析器

我们需要将es模板放在第一个，因为它相比能够匹配所有字符串字段的en模板更加具体：

```json
PUT /my_index
{
    "mappings": {
        "my_type": {
            "dynamic_templates": [
                { "es": {
                      "match":              "*_es", 
                      "match_mapping_type": "string",
                      "mapping": {
                          "type":           "string",
                          "analyzer":       "spanish"
                      }
                }},
                { "en": {
                      "match":              "*", 
                      "match_mapping_type": "string",
                      "mapping": {
                          "type":           "string",
                          "analyzer":       "english"
                      }
                }}
            ]
}}}
```

match_mapping_type允许你只对特定类型的字段使用模板，正如标准动态映射规则那样，比如string，long等。

match参数只会匹配字段名，path_match参数用于匹配对象中字段的完整路径，比如address.*.name可以匹配如下字段：

```json
{
    "address":
        "city":
            "name": "New York"
        }
    }
}
```

unmatch和path_unmatch模式能够用来排除某些字段，没有被排除的字段则会被匹配。

更多的配置选项可以在[根对象的参考文档](http://www.elasticsearch.org/guide/en/elasticsearch/reference/1.4//mapping-root-object-type.html)中找到。
