---
title: ES集群配置
date: 2021-01-13 19:39:32
categories: ELK
tags: es
---

# 前置操作

需要确认其它es节点中的data目录，一定要清空，不能有数据。

# 配置集群

修改`elasticsearch.yml`这个配置文件如下：

```bash
# 配置集群名称，保证每个节点的名称相同，如此就能都处于一个集群之内了
cluster.name: es-cluster

# 每一个节点的名称，必须不一样
node.name: es-node1

# http端口（使用默认即可）
http.port: 9200

# 主节点，作用主要是用于来管理整个集群，负责创建或删除索引，管理其他非master节点（相当于企业老总）
node.master: true

# 数据节点，用于对文档数据的增删改查
node.data: true

# 集群列表
discovery.seed_hosts: ["192.168.1.184", "192.168.1.185", "192.168.1.186"]

# 启动的时候使用一个master节点
cluster.initial_master_nodes: ["es-node1"]
```

最后可以通过如下命令查看配置文件的内容：

```bash
more elasticsearch.yml | grep ^[^#]
```