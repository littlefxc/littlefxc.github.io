---
title: kafka设置某个topic的数据过期时间
date: 2019-08-22 09:24:49
categories: mq
tags: 
- mq
- kafka
---

#  kafka 单独设置某个topic的数据过期时间

kafka 默认存放7天的临时数据，如果遇到磁盘空间小，存放数据量大，可以设置缩短这个时间。

## 全局设置

修改 server.properties

```yaml
log.retention.hours=72
log.cleanup.policy=delete
```

## 单独对某一个topic设置过期时间

但如果只有某一个topic数据量过大。
想单独对这个topic的过期时间设置短点：

```
./kafka-configs.sh --zookeeper localhost:2181 --alter --entity-name mytopic --entity-type topics --add-config retention.ms=86400000
```

retention.ms=86400000 为一天，单位是毫秒。

查看设置：

```sh
$ ./kafka-configs.sh --zookeeper localhost:2181 --describe --entity-name mytopic --entity-type topics
Configs for topics:wordcounttopic are retention.ms=86400000
```

## 立即删除某个topic下的数据

```sh
./kafka-topics.sh --zookeeper localhost:2181 --alter --topic mytopic --config cleanup.policy=delete
```


