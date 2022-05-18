---
title: ES 深度分页 & 滚动搜索
date: 2021-01-12 21:17:45
categories: ELK
tags: es
---

# 分页查询

```jsx
POST /shop/_doc/_search
{
    "query": {
        "match_all": {}
    },
    "from": 0,
    "size": 10
}                                                                
```

# 深度分页

![https://gitee.com/littlefxc/oss/raw/master/images/es-13_1.png](https://gitee.com/littlefxc/oss/raw/master/images/es-13_1-20210112211825281.png)

深度分页其实就是搜索的深浅度，比如第1页，第2页，第10页，第20页，是比较浅的；第10000页，第20000页就是很深了。

使用如下操作：

```jsx
{
    "query": {
        "match_all": {}
    },
    "from": 9990,
    "size": 10
}

{
    "query": {
        "match_all": {}
    },
    "from": 9999,
    "size": 10
}
```

我们在获取第9999条到10009条数据的时候，其实每个分片都会拿到10009条数据，然后集合在一起，总共是10009*3=30027条数据，针对30027数据再次做排序处理，最终会获取最后10条数据。

如此一来，搜索得太深，就会造成性能问题，会耗费内存和占用cpu。而且es为了性能，他不支持超过一万条数据以上的分页查询。那么如何解决深度分页带来的性能呢？其实我们应该避免深度分页操作（限制分页页数），比如最多只能提供100页的展示，从第101页开始就没了，毕竟用户也不会搜的那么深，我们平时搜索淘宝或者百度，一般也就看个10来页就顶多了。

譬如淘宝搜索限制分页最多100页，如下：

![https://gitee.com/littlefxc/oss/raw/master/images/es-13_2.jpg](https://gitee.com/littlefxc/oss/raw/master/images/es-13_2.jpg)

# 通过设置index.max_result_window来突破10000数据

```jsx
# 查询shop索引的设置
GET /shop/_settings
# 修改 shop 索引的最大返回结果
PUT /shop/_settings
{ 
    "index.max_result_window": "20000"
}
```

# scroll 滚动搜索

一次性查询1万+数据，往往会造成性能影响，因为数据量太多了。这个时候可以使用滚动搜索，也就是 `scroll`。滚动搜索可以先查询出一些数据，然后再紧接着依次往下查询。在第一次查询的时候会有一个滚动id，相当于一个`锚标记`，随后再次滚动搜索会需要上一次搜索的`锚标记`，根据这个进行下一次的搜索请求。每次搜索都是基于一个历史的数据快照，查询数据的期间，如果有数据变更，那么和搜索是没有关系的，搜索的内容还是快照中的数据。

- scroll=1m，相当于是一个session会话时间，搜索保持的上下文时间为1分钟。

```jsx
POST /shop/_search?scroll=1m
{
    "query": { 
    	"match_all": {
    	}
    },  
    "sort" : ["_doc"], 
    "size":  5
}

POST /_search/scroll
{
    "scroll": "1m", 
    "scroll_id" : "your last scroll_id"
}
```

- 官文地址：https://www.elastic.co/guide/cn/elasticsearch/guide/current/scroll.html

# mget 批量查询

[26、ES中使用mget批量查询api（学习笔记，来自课程资料 + 自己整理）_涂作权的博客-CSDN博客_es mget](https://blog.csdn.net/tototuzuoquan/article/details/80558611)