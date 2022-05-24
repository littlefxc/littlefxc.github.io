---
title: Consul 单机集群搭建
date: 2021-01-11 16:41:37
categories: 注册中心
tags: consul
---

# 1 Consul 单机集群搭建

本文是在生产环境中Consul服务端节点不稳定后，导致注册到Consul集群上的部分服务不可用，从而造成开放能力平台无法使用的背景下产生的。

目的是探讨一个高可用Consul集群方案。

## 1.1 consul 架构

Server负责组成 cluster 的复杂工作（选举、状态维护、转发请求到 lead），以及 consul 提供的服务（响应 RCP 请求）。考虑到容错和收敛，一般部署 3 ~ 5 个比较合适，而client数量不做限制，架构如下:

![consul 架构](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/consul架构1.png)

## 1.2 Consul 节点规划

| **节点名称**    | 节点类型 | **HTTP端口** | **DNS端口** | **serf_lan端口** | **serf_wan端口** |
| --------------- | -------- | ------------ | ----------- | ---------------- | ---------------- |
| consul-server-1 | server   | 8501         | 8601        | 8001             | 8002             |
| consul-server-2 | server   | 8502         | 8602        | 8101             | 8102             |
| consul-server-3 | server   | 8503         | 8603        | 8201             | 8202             |
| consul-client-1 | client   | 8504         | 8604        | 8301             | 8302             |
| consul-client-2 | client   | 8505         | 8605        | 8401             | 8402             |

## 1.3 Consul 服务端节点配置文件模版

```json
{
    "bind_addr": "127.0.0.1",
    "client_addr": "127.0.0.1",
    "ports": {
        "http": 8501,
        "dns": 8601,
        "serf_lan": 8001,
        "serf_wan": 8002,
        "server": 8000
    },
    "datacenter": "dc1",
    "data_dir": "/Users/fengxuechao/WorkSpace/IdeaProjects/consul/consul-server-1/data",
    "log_level": "INFO",
    "log_file": "/Users/fengxuechao/WorkSpace/IdeaProjects/consul/consul-server-1/log/consul.log",
    "node_name": "consul-server-1",
    "disable_host_node_id": true,
    "server": true,
    "ui": true,
    "bootstrap_expect": 3,
    "rejoin_after_leave": true,
    "retry_join": [
        "127.0.0.1:8001",
        "127.0.0.1:8101",
        "127.0.0.1:8201"
    ]
}
```

注意：

- **disable_host_node_id**：不使用host信息生成node ID，适用于同一台服务器部署多个实例用于测试的情况。随机生成nodeID。

## 1.4 Consul 客户端节点配置文件模版

```json
{
    "bind_addr": "127.0.0.1",
    "client_addr": "127.0.0.1",
    "ports": {
        "http": 8505,
        "dns": 8605,
        "serf_lan": 8401,
        "serf_wan": 8402,
        "server": 8400
    },
    "datacenter": "dc1",
    "data_dir": "/Users/fengxuechao/WorkSpace/IdeaProjects/consul/consul-client-2/data",
    "log_level": "INFO",
    "log_file": "/Users/fengxuechao/WorkSpace/IdeaProjects/consul/consul-client-2/log/consul.log",
    "node_name": "consul-client-2",
    "disable_host_node_id": true,
    "server": false,
    "ui": true,
    "rejoin_after_leave": true,
    "retry_join": [
        "127.0.0.1:8001",
        "127.0.0.1:8101",
        "127.0.0.1:8201"
    ]
}
```

## 1.5 搭建集群

```sh
# 创建 Consul 服务端节点目录
mkdir consul-server-{1,2,3}
# 创建 Consul 服务端节点可执行文件目录
mkdir consul-server-{1,2,3}/bin
# 创建  Consul 服务端节点配置文件目录
mkdir consul-server-{1,2,3}/config
# 创建  Consul 服务端节点数据文件目录
mkdir consul-server-{1,2,3}/data
# 创建  Consul 服务端节点日志文件目录
mkdir consul-server-{1,2,3}/log

# consul 客户端节点目录结构与服务端节点目录结构保持一致
...

# Consul 服务端节点搭建
nohup consul-server-1/bin/consul agent -config-dir=consul-server-1/config &
nohup consul-server-2/bin/consul agent -config-dir=consul-server-2/config &
nohup consul-server-3/bin/consul agent -config-dir=consul-server-3/config &
# Consul 客户端节点搭建
nohup consul-client-1/bin/consul agent -config-dir=consul-client-1/config &
nohup consul-client-2/bin/consul agent -config-dir=consul-client-2/config &
```

# 2 Consul 的一些命令

## 2.1 查看 consul 集群列表

验证 Consul 集群是否搭建成功

```sh
consul-server-1/bin/consul operator raft list-peers -http-addr=127.0.0.1:8501
```

## 2.2 查看 Consul 节点列表

查看 Consul 的所有节点列表

```sh
consul-server-1/bin/consul members -http-addr=127.0.0.1:8501
```

## 2.3 移除一个 Server 节点

验证 Consul 客户端节点不会丢失

```sh
consul-server-1/bin/consul leave -http-addr=127.0.0.1:8501
```

# 3 验证

## 3.1 模拟一个 Consul 服务端宕机

1. 移除 Consul 服务端节点前：

![image-20210112101929745](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210112101929745.png)

2. 移除 Consul 服务端节点。命令参考章节2.3。

   结果如下：

   ![image-20210112102147403](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210112102147403.png)

   ![image-20210112102223434](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210112102223434.png)

# 4 结论

说明当一个consul服务端节点宕机，并不会影响 Consul 客户端节点的可用性。

在每个数据中心，client和server是混合的。一般建议有3-5台server。这是基于有故障情况下的可用性和性能之间的权衡结果，因为越多的机器加入达成共识越慢。虽然每个节点的服务注册数量是有上限的，但是并不限制client的数量，它们可以很容易的扩展到数千或者数万台。



# 5 参考资源

https://blog.csdn.net/u014635374/article/details/106313858