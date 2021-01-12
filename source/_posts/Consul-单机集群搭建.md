---
title: Consul 单机集群搭建
date: 2021-01-11 16:41:37
categories: 注册中心
tags: consul
---

# Consul 单机集群搭建

## consul 架构

官方比较推荐的是三台以上的server，而client数量不做限制，网上大多数推荐的架构如下:

![consul 架构](https://gitee.com/littlefxc/oss/raw/master/images/consul架构1.png)

## Consul 节点规划

| **节点名称**    | 节点类型 | **HTTP端口** | **DNS端口** | **serf_lan端口** | **serf_wan端口** |
| --------------- | -------- | ------------ | ----------- | ---------------- | ---------------- |
| consul-server-1 | server   | 8501         | 8001        | 8002             | 8000             |
| consul-server-2 | server   | 8502         | 8101        | 8102             | 8100             |
| consul-server-3 | server   | 8503         | 8201        | 8103             | 8200             |
| consul-client-1 | client   | 8504         | 8301        | 8104             | 8300             |
| consul-client-2 | client   | 8505         | 8401        | 8105             | 8400             |

## Consul 服务端节点配置文件模版

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

- **disable_host_node_id**：禁用主机信息生成节点ID。因为是单机，所以要禁用它。单机集群搭建成功与否就靠它。

## Consul 客户端节点配置文件模版

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

## 搭建集群

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

# Consul 的一些命令

## 查看 consul 集群列表

验证 Consul 集群是否搭建成功

```sh
consul-server-1/bin/consul operator raft list-peers -http-addr=127.0.0.1:8501
```

## 查看 Consul 节点列表

查看 Consul 的所有节点列表

```sh
consul-server-1/bin/consul members -http-addr=127.0.0.1:8501
```

## 移除一个 Server 节点

验证 Consul 客户端节点不会丢失

```sh
consul-server-1/bin/consul leave -http-addr=127.0.0.1:8501
```

# 参考资源

https://blog.csdn.net/u014635374/article/details/106313858