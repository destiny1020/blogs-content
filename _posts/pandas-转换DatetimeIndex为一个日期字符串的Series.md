---
title: '[pandas] 转换DatetimeIndex为一个日期字符串的Series'
date: 2015-09-24 00:34:00
categories: ['工具, 库与框架', pandas]
tags: [Python, pandas]
---

## 遇到的问题

需要将一个DatetimeIndex转换为一个日期字符串的Series类型。

比如，有一个DatetimeIndex是这样的：

```python
print dti
DatetimeIndex(['2015-09-21 10:30:00', '2015-09-21 11:30:00',
               '2015-09-21 14:00:00', '2015-09-21 15:00:00',
               '2015-09-22 10:30:00', '2015-09-22 11:30:00',
               '2015-09-22 14:00:00', '2015-09-22 15:00:00'],
              dtype='datetime64[ns]', freq=None, tz=None)
```

现在只需要这些数据里面的2015-09-21，2015-09-22这样的日期信息，时间信息可以省略。

<!-- More -->

---

## 解决方案

如果Index的类型是普通的pandas.core.index.Index，那么这个问题好解决：

```python
df.index.str.split(' ').str[0]
```

然而当尝试在DatetimeIndex上使用str对象时，会抛出异常：

```
AttributeError: Can only use .str accessor with string values (i.e. inferred_typ
e is 'string', 'unicode' or 'mixed')
```

经过查看API，发现可以先将DatetimeIndex转换为一个类型为datetime的数组，然后对该数组进行操作得到一个numpy.ndarray，最后将这个array转化为Series即可，具体代码如下所示：

```python
pydate_array = dti.to_pydatetime()
date_only_array = np.vectorize(lambda s: s.strftime('%Y-%m-%d'))(pydate_array )
date_only_series = pd.Series(date_only_array)
```

最后得到的结果就是只含有日期的Series：

```python
print date_only_series 

0    2015-09-21
1    2015-09-21
2    2015-09-21
3    2015-09-21
4    2015-09-22
5    2015-09-22
6    2015-09-22
7    2015-09-22
dtype: object
```

