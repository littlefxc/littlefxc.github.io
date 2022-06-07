---
title: 安装kafka集群
date: 2021-04-07 19:41:28
tags:
---

# 安装kafka集群

## 下载kafka

#kafka官网：
http://kafka.apache.org/
http://kafka.apache.org/downloads

<!-- more -->

## 配置 server.properties

```
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# see kafka.server.KafkaConfig for additional details and defaults

############################# Server Basics #############################

# The id of the broker. This must be set to a unique integer for each broker.
#每个broker在集群中的唯一标识，不能重复
broker.id=0
#端口
port=9092
#broker主机地址
host.name=server1

############################# Socket Server Settings #############################

# The address the socket server listens on. It will get the value returned from
# java.net.InetAddress.getCanonicalHostName() if not configured.
#   FORMAT:
#     listeners = listener_name://host_name:port
#   EXAMPLE:
#     listeners = PLAINTEXT://your.host.name:9092
#listeners=PLAINTEXT://:9092

# Hostname and port the broker will advertise to producers and consumers. If not set,
# it uses the value for "listeners" if configured.  Otherwise, it will use the value
# returned from java.net.InetAddress.getCanonicalHostName().
#advertised.listeners=PLAINTEXT://your.host.name:9092

# Maps listener names to security protocols, the default is for them to be the same. See the config documentation for more details
#listener.security.protocol.map=PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT,SASL_SSL:SASL_SSL

# The number of threads that the server uses for receiving requests from the network and sending responses to the network
#broker处理消息的线程数
num.network.threads=3

# The number of threads that the server uses for processing requests, which may include disk I/O
#broker处理磁盘io的线程数
num.io.threads=8

# The send buffer (SO_SNDBUF) used by the socket server
#socket发送数据缓冲区
socket.send.buffer.bytes=102400

# The receive buffer (SO_RCVBUF) used by the socket server
#socket接收数据缓冲区
socket.receive.buffer.bytes=102400

# The maximum size of a request that the socket server will accept (protection against OOM)
#socket接收请求最大值
socket.request.max.bytes=104857600


############################# Log Basics #############################

# A comma seperated list of directories under which to store log files
#kafka数据存放目录位置，多个位置用逗号隔开
log.dirs=/usr/local/kafka/kafka_2.11-1.0.0/kfk-logs

# The default number of log partitions per topic. More partitions allow greater
# parallelism for consumption, but this will also result in more files across
# the brokers.
#topic默认的分区数
num.partitions=1

# The number of threads per data directory to be used for log recovery at startup and flushing at shutdown.
# This value is recommended to be increased for installations with data dirs located in RAID array.
#恢复线程数
num.recovery.threads.per.data.dir=1

############################# Internal Topic Settings  #############################
# The replication factor for the group metadata internal topics "__consumer_offsets" and "__transaction_state"
# For anything other than development testing, a value greater than 1 is recommended for to ensure availability such as 3.
#默认副本数
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1

############################# Log Flush Policy #############################

# Messages are immediately written to the filesystem but by default we only fsync() to sync
# the OS cache lazily. The following configurations control the flush of data to disk.
# There are a few important trade-offs here:
#    1. Durability: Unflushed data may be lost if you are not using replication.
#    2. Latency: Very large flush intervals may lead to latency spikes when the flush does occur as there will be a lot of data to flush.
#    3. Throughput: The flush is generally the most expensive operation, and a small flush interval may lead to exceessive seeks.
# The settings below allow one to configure the flush policy to flush data after a period of time or
# every N messages (or both). This can be done globally and overridden on a per-topic basis.

# The number of messages to accept before forcing a flush of data to disk
#log.flush.interval.messages=10000

# The maximum amount of time a message can sit in a log before we force a flush
#log.flush.interval.ms=1000

############################# Log Retention Policy #############################

# The following configurations control the disposal of log segments. The policy can
# be set to delete segments after a period of time, or after a given size has accumulated.
# A segment will be deleted whenever *either* of these criteria are met. Deletion always happens
# from the end of the log.

# The minimum age of a log file to be eligible for deletion due to age
#消息日志最大存储时间，这里是7天
log.retention.hours=168

# A size-based retention policy for logs. Segments are pruned from the log unless the remaining
# segments drop below log.retention.bytes. Functions independently of log.retention.hours.
#log.retention.bytes=1073741824

# The maximum size of a log segment file. When this size is reached a new log segment will be created.
#每个日志段文件大小，这里是1g
log.segment.bytes=1073741824

# The interval at which log segments are checked to see if they can be deleted according
# to the retention policies
#消息日志文件大小检查间隔时间
log.retention.check.interval.ms=300000

############################# Zookeeper #############################

# Zookeeper connection string (see zookeeper docs for details).
# This is a comma separated host:port pairs, each corresponding to a zk
# server. e.g. "127.0.0.1:3000,127.0.0.1:3001,127.0.0.1:3002".
# You can also append an optional chroot string to the urls to specify the
# root directory for all kafka znodes.
#zookeeper集群地址
zookeeper.connect=server1:2181,server2:2181,server3:2181

# Timeout in ms for connecting to zookeeper
#zookeeper连接超时时间
zookeeper.connection.timeout.ms=6000


############################# Group Coordinator Settings #############################

# The following configuration specifies the time, in milliseconds, that the GroupCoordinator will delay the initial consumer rebalance.
# The rebalance will be further delayed by the value of group.initial.rebalance.delay.ms as new members join the group, up to a maximum of max.poll.interval.ms.
# The default value for this is 3 seconds.
# We override this to 0 here as it makes for a better out-of-the-box experience for development and testing.
# However, in production environments the default value of 3 seconds is more suitable as this will help to avoid unnecessary, and potentially expensive, rebalances during application startup.
group.initial.rebalance.delay.ms=0
```

## 修改其它节点的配置文件

### server2节点

```
cd /usr/local/kafka/kafka_2.11-1.0.0/config

vim server.properties
----------------------------------------------------
# The id of the broker. This must be set to a unique integer for each broker.
broker.id=1
port=9092
```

### server3节点

```
cd /usr/local/kafka/kafka_2.11-1.0.0/config

vim server.properties
----------------------------------------------------
# The id of the broker. This must be set to a unique integer for each broker.
broker.id=2
port=9092
```

# 启动kafka集群

```sh
# 分别在三台节点执行：node01/node02/node03
## 启动kafka集群-daemon(以后台服务方式启动) 后面跟的是以配置文件启动
/usr/local/kafka/kafka_2.11-1.0.0/bin/kafka-server-start.sh -daemon /usr/local/kafka/kafka_2.11-1.0.0/config/server.properties

 ## 停止kafka集群
/usr/local/kafka/kafka_2.11-1.0.0/bin/kafka-server-stop.sh
```

# zookeeper 查看 kafka 集群

```sh
zkCli.sh
ls /brokers/ids
```

结果

```
[1, 2, 3]
```

说明正确kafka正确启动了