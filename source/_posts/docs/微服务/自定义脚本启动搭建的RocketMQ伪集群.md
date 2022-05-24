---
title: "自定义脚本启动搭建的RocketMQ伪集群"
date: 2022-04-02T10:47:11+08:00
categories: 分布式系统
tags:
- rocketmq
---

## 引言

工欲善其事，必先利其器。

因为只有一台电脑，只能搭建伪集群来学习了，但是，本身又是个偷懒的人，启动伪集群 RocketMQ 的命令有点多，不想敲那么多的命令，顺便将搭建 RocketMQ 集群的部署方式记录一下。

RocketMQ 的部署方式有3种：

- 2m-noslave：多 Master 模式，无 Slave。[双主模式]
  - 优点：配置简单，性能最高
  - 缺点：可能会有少量消息丢失（配置相关），单台机器重启或宕机期间，该机器下未被消费的消息在机器恢复前不可订阅，影响消息实时性
- 2m-2s-sync：多 Master 多 Slave 模式，同步双写。[双主双从+同步模式]
  - 优点：服务可用性与数据可用性非常高
  - 缺点：性能比异步集群略低
- 2m-2s-async：多 Master 多 Slave 模式，异步复制。[双主双从+异步模式]
  - 优点：性能同多Master几乎一样，实时性高，主备间切换对应用透明，不需人工干预
  - 缺点：Master宕机或磁盘损坏时会有少量消息丢失

本文主要记录一下同步双写模式的搭建步骤和一键启动脚本。

## 端口规划

首先，因为只有一台电脑，端口要进行规划，以避免端口占用的问题。

采用的方案就是“双主双从+同步模式”。

| 名称           | 端口  |
| -------------- | ----- |
| namesrv1       | 9876  |
| Namesrv2       | 9877  |
| brokera-master | 10910 |
| brokera-slave  | 10920 |
| brokerb-master | 10930 |
| brokerb-slave  | 10940 |

## 下载

https://www.apache.org/dyn/closer.cgi?path=rocketmq/4.9.3/rocketmq-all-4.9.3-bin-release.zip

## 配置文件

```shell
vim rocketmq/conf/2m-2s-sync/broker-a.properties
```

```yaml
# 集群名字
brokerClusterName=DefaultCluster
# broker 名字，可重复，master 的名字和 slave 的名字保持一致
brokerName=broker-a
# 0 表示 master, >0 表示 slave 
brokerId=0
# 删除文件时间点，默认凌晨 4点
deleteWhen=04
# 文件保留时间，默认 48 小时
fileReservedTime=48
# Broker 的角色
brokerRole=SYNC_MASTER
# 刷盘方式
flushDiskType=ASYNC_FLUSH
# Broker 对外服务的监听端口, 注意需改端口, 并且要和默认的10911相差5以上
listenPort=10910
# 是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
# 是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
# nameServer地址，分号分割
namesrvAddr=localhost:9876;localhost:9877
# 存储路径
storePathRootDir=/usr/local/var/lib/rocketmq/broker-a
# commitLog 存储路径
storePathCommitLog=/usr/local/var/lib/rocketmq/broker-a/commitlog
# 消费队列存储路径存储路径
storePathConsumeQueue=/usr/local/var/lib/rocketmq/broker-a/consumequeue
# 消息索引存储路径
storePathIndex=/usr/local/var/lib/rocketmq/broker-a/index
# checkpoint 文件存储路径
storeCheckpoint=/usr/local/var/lib/rocketmq/broker-a/checkpoint
# abort 文件存储路径
abortFile=/usr/local/var/lib/rocketmq/broker-a/about
```

其它节点的配置文件类似只需修改一下 `brokerId`、 `brokerName`、`listenPort`、`brokerRole`和`store*`这些参数即可。

## 修改 JVM 启动资源要素

还是因为本身只有一台机器，资源有限的原因。

```shell
vim rocketmq-4.9.3/bin/runbroker.sh
```

![image-20220402143803612](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220402143803612.png)

```shell
vim rocketmq-4.9.3/bin/runserver.sh
```

![image-20220402143514440](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220402143514440.png)

## 一键启动脚本

```bash
#!/bin/bash
export JAVA_HOME="/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home"
export ROCKETMQ_HOME="/usr/local/rocketmq-4.9.3"
export ROCKETMQ_LOG_DIR="/usr/local/var/log/rocketmq"

if [ $1 == "startall" ] then
    # 启动 namesrv1
    nohup sh ${ROCKETMQ_HOME}/bin/mqnamesrv -c ${ROCKETMQ_HOME}/conf/2m-2s-sync/namesrv-1.properties > ${ROCKETMQ_LOG_DIR}/mqnamesrv1.log 2>&1 &
    
    # 启动 namesrv2
    nohup sh ${ROCKETMQ_HOME}/bin/mqnamesrv -c ${ROCKETMQ_HOME}/conf/2m-2s-sync/namesrv-2.properties > ${ROCKETMQ_LOG_DIR}/mqnamesrv2.log 2>&1 &

    # 启动 broker-a
    nohup sh ${ROCKETMQ_HOME}/bin/mqbroker -c ${ROCKETMQ_HOME}/conf/2m-2s-sync/broker-a.properties > ${ROCKETMQ_LOG_DIR}/broker-a.log 2>&1 &

    # 启动 broker-a-s
    nohup sh ${ROCKETMQ_HOME}/bin/mqbroker -c ${ROCKETMQ_HOME}/conf/2m-2s-sync/broker-a-s.properties > ${ROCKETMQ_LOG_DIR}/broker-a-s.log 2>&1 &

    # 启动 broker-b
    nohup sh ${ROCKETMQ_HOME}/bin/mqbroker -c ${ROCKETMQ_HOME}/conf/2m-2s-sync/broker-b.properties > ${ROCKETMQ_LOG_DIR}/broker-b.log 2>&1 &

    # 启动 broker-b-s
    nohup sh ${ROCKETMQ_HOME}/bin/mqbroker -c ${ROCKETMQ_HOME}/conf/2m-2s-sync/broker-b-s.properties > ${ROCKETMQ_LOG_DIR}/broker-b-s.log 2>&1 &

    export NAMESRV_ADDR=localhost:9876,localhost:9877

    echo "rocketmq 集群启动成功!"
elif[ $1 == "start" ] then 
    nohup ${ROCKETMQ_HOME}/bin/mqnamesrv &
    nohup ${ROCKETMQ_HOME}/bin/mqbroker -n localhost:9876 &
    export NAMESRV_ADDR=localhost:9876
    echo "rocketmq 单机启动成功!"
elif [ $1 == "stop" ] then 
    # 先关闭 broker
    sh ${ROCKETMQ_HOME}/bin/mqshutdown broker
    # 再关闭 namesrv
    sh ${ROCKETMQ_HOME}/bin/mqshutdown namesrv
    echo "rocketmq 关闭成功!"
else 
    echo "rockermq_service.sh (startall|start|stop)"
fi
```

## 运行情况

![image-20220402163857315](/Users/fengxuechao/Library/Application%20Support/typora-user-images/image-20220402163857315.png)

## 参考文档

[官方中文部署文档](https://github.com/apache/rocketmq/blob/master/docs/cn/operation.md)
