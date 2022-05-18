---
title: Python中==和is的区别
date: 2020-06-18 19:52:06
categories: Python
tags: 
---

## 前言

本文主要内容是 Python 中运算符 `==` 和 `is` 的区别。

进入正文前，首先简单介绍一下 Python 中对象的 3 个基本要素，id(身份标识)、type(数据类型)和value(值)。


## 正文

`==` 是python标准操作符中的比较操作符，用来比较判断两个对象的value(值)是否相等，例如下面两个字符串间的比较：

```python
>>> a = 'cheesezh'
>>> b = 'cheesezh'
>>> a == b
True
```

`is` 也被叫做同一性运算符，这个运算符比较判断的是对象间的唯一身份标识，也就是id是否相同。

```python
>>> x = y = [4,5,6]
>>> z = [4,5,6]
>>> x == y
True
>>> x == z
True
>>> x is y
True
>>> x is z
False
>>>
>>> print id(x)
3075326572
>>> print id(y)
3075326572
>>> print id(z)
3075328140
```

可以明显的看到前 3 个比较都是 True， 最后一个是 False。

使用 `id()` 方法查看 x, y, z 的对象ID就明白了。

我在这里使用的是数组，其实，当它们是 tuple, list, dict 或者 set 时也一样。

不过，当类型是 int 或者 string 时，它们的对象ID都会一样,

```python
>>> a = 1
>>> b = 1
>>> a is b
True
>>> a = "asd"
>>> b = "asd"
>>> a is b
True
```