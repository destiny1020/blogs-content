---
title: '[Jackson] 使用ObjectMapper对含有任意key的JSON进行反序列化'
date: 2015-07-05 17:42:00
categories: ['工具, 库与框架', Jackson]
tags: [Java, Jackson, 反序列化]
---

## 使用ObjectMapper对含有任意key的JSON进行反序列化

在调用某个RESTful API后，返回的JSON字符串中含有没有预先定义的key，和结构固定的JSON相比，它需要一些额外的操作。

对于结构固定的JSON，使用ObjectMapper结合某个预先定义的实体类型可以非常方便地完成反序列化工作，比如对下面这样的JSON：

```json
GET /person/1

{
	"id": "1",
	"name": "dm_vincent",
	"age": "28"
}
```

<!-- More -->

结合一个实体类型，可以很轻松的完成反序列化工作：
```java
public class Person {
	public long id;
	public String name;
	public int age;
}
```

```java
public static <T> T getEntity(String jsonString, Class<T> prototype) {

    ObjectMapper objectMapper = new ObjectMapper();
    try {
      return (T) objectMapper.readValue(jsonString, prototype);
    } catch (IOException e) {
      e.printStackTrace();
    }

    return null;
  }
```

但是在某些支持一次性获取多条记录的API中，就出现问题了。比如拥有下面这种格式的API：
```json
GET /person/1,2,3

{
	"dm_vincent": {
		"id": "1",
		"name": "dm_vincent",
		"age": "28"
	},
	"dm_vincent2": {
		"id": "2",
		"name": "dm_vincent2",
		"age": "29"
	},
	"dm_vincent3": {
		"id": "3",
		"name": "dm_vincent3",
		"age": "30"
	}
}
```

虽然需要获取的实体类型还是那个Person类，可是这个时候就不像上面那样简单明了了。比如下面这段代码在反序列化的时候会出现问题：

```java
public class PersonWrapper {
	public Map<String, Person> persons;
}
```

我们的意图是明确的，将返回的多个Person实体对象放到一个Map结构中。但是问题就在于返回的JSON中的keys是不固定的(比如上述JSON中的keys是人名)，这导致反序列化失败。毕竟默认配置下的ObjectMapper也没有聪明到这种程度，能够猜测你是想要将多个实体放到Map中。

正确的做法之一是使用ObjectMapper的readTree方法：

```java
public static <T> EntityWrapper<T> getEntityWrapper(String jsonString, Class<T> prototype) {
    ObjectMapper objectMapper = new ObjectMapper();
    EntityWrapper<T> wrapper = new EntityWrapper<T>();
    try {
      JsonNode root = objectMapper.readTree(jsonString);
      Iterator<Entry<String, JsonNode>> elements = root.fields();

      while (elements.hasNext()) {
        Entry<String, JsonNode> node = elements.next();
        String key = node.getKey();
        T element = objectMapper.readValue(node.getValue().toString(), prototype);
        wrapper.addEntry(key, element);
      }

      return wrapper;
    } catch (IOException e) {
      e.printStackTrace();
    }

    return null;
  }
```

简单解释一下上述代码：

使用`root.field()`方法能够得到返回的JSON中的所有key-value对。
然后循环提取某个键值对的key和value，对于value我们可以直接使用之前的策略进行反序列化，因为这部分的结构也是固定的。

------
## 忽略不需要的字段

有时候，返回的JSON字符串中含有我们并不需要的字段，那么当对应的实体类中不含有该字段时，会抛出一个异常，告诉你有些字段没有在实体类中找到。解决办法很简单，在声明ObjectMapper之后，加上上述代码：

```java
ObjectMapper objectMapper = new ObjectMapper();
objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
```
