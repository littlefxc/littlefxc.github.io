---
title: Consul使用记录
date: 2019-05-23 09:53:08
categories: 注册中心
tags:
- consul
---

## 1. consul 集群模式

```sh
consul agent -server -data-dir=./data/ -node=s1 -bind=192.168.217.86 -ui -rejoin -client=0.0.0.0 -bootstrap-expect=1 -ui
```

参数介绍：

* -server 表示是以服务端身份启动，去掉这个参数表示 `client` 模式
* -bind 表示绑定到哪个ip（有些服务器会绑定多块网卡，可以通过bind参数强制指定绑定的ip），一般是本机IP
* -client 指定客户端访问的ip(consul有丰富的api接口，这里的客户端指浏览器或调用方)，0.0.0.0表示不限客户端ip
* -bootstrap-expect=3 表示server集群最低节点数为3，低于这个值将工作不正常(注：类似zookeeper一样，通常集群数为奇数，方便选举，consul采用的是raft算法)
* -data-dir 表示指定数据的存放(持久化)目录（该目录必须存在）
* -node 表示节点在web ui中显示的名称

启动成功后，终端窗口不要关闭，可以在浏览器里，访问下，类似 http://192.168.212.73:8500/， 正常的话，可以正常打开web界面。

为了防止终端关闭后，consul退出，可以在刚才命令上，加点东西，类似：`nohup xxx  > /dev/null 2>&1 & `

### 1.1. 命令行建立 consul 集群

服务端1：

```sh
consul agent -server -bootstrap-expect=2 -data-dir=./data/ -bind=192.168.217.134 -client=0.0.0.0 -node=s1 -ui
```

服务端2：

```sh
consul agent -server -bootstrap-expect=2 -data-dir=./data/ -bind=192.168.217.72 -client=0.0.0.0 -join 192.168.217.134 -node=s2 -ui
```

服务端3：

```sh
consul agent -server -bootstrap-expect=2 -data-dir=./data/ -bind=192.168.217.86 -client=0.0.0.0 -join 192.168.217.134 -node=s3 -ui
```

客户端1：

```sh
consul agent -data-dir=./data/ -bind=192.168.217.87 -client=0.0.0.0 -join 192.168.217.134 -node=c1 -ui
```

### 1.2. 配置文件方式建立 consul 集群

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190522164211256.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xpdHRsZV9meGM=,size_16,color_FFFFFF,t_70)

例：

```json
{
  /* 表示server集群最低节点数 */
  "bootstrap_expect": 2,
  /* 表示该节点绑定的地址, 默认0.0.0.0 */
  "bind_addr": "192.168.217.86",
  /* 访问该节点的地址, 默认127.0.0.1*/
  "client_addr": "0.0.0.0",
  /* 访问该节点的数据中心*/
  "datacenter": "data-center-1",
  /* 必须, 数据保存的地址*/
  "data_dir": "/home/awifi/consul-server/data",
  "addresses": {
    /* 表示该节点绑定的dns地址, 以空格分隔的要绑定的地址列表*/
    "dns": "192.168.217.86",
    /* 表示该节点绑定的http地址, 以空格分隔的要绑定的地址列表*/
    "http": "192.168.217.86 127.0.0.1"
  },
  /* 节点名，必须*/
  "node_name": "server:192.168.212.73:8500",
  "rejoin_after_leave": true,
  /* 表示该节点类型是server*/
  "server": true,
  /* 允许在第一次尝试失败时重试连接 */
"retry_join": [
        "192.168.217.72",
        "192.168.217.134"
    ],
  /* 表示开启web界面*/
  "ui": true
}
```

然后在另一台机器上执行
./consul join 192.168.217.86
或者可以加入配置

```json
{
  "start_join":[],
  "retry_join":[]
}
```

`start_join`：等同于 命令行参数` -join` : 表示启动时加入的集群地址
`retry_join`：等同于 命令行参数` -retry-join` : 允许在第一次尝试失败时重试连接，该列表可以包含IPv4，IPv6或DNS地址。

#### 1.2.1.  为什么 http 地址要绑定 127.0.0.1 ？

如果不绑定127.0.0.1，在该服务器上就会无法使用除 ./consul agent 以外的命令，

例如:
组建集群模式的核心命令：`consul join 192.168.217.86`
优雅的注销节点的命令：  `consul leave`
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190522164607754.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0xpdHRsZV9meGM=,size_16,color_FFFFFF,t_70)

## 2. 查看集群状态

```sh
consul operator raft list-peers
```

 运行结果：

![查看集群状态](https://img-blog.csdnimg.cn/20190522162138210.png)

运行结果红色方框部分可以看出集群模式中谁是 `leader`