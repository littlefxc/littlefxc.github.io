---
title: flink集群安装
date: 2021-04-08 16:11:55
categories: 大数据
tags:
- flink
---

# 1 前言

Flink 是一个以 Java 及 Scala 作为开发语言的开源大数据项目，代码开源在 GitHub 上，并使用 Maven 来编译和构建项目。

关于开发测试环境，Mac OS、Linux 系统或者 Windows 都可以。如果使用的是 Windows 10 系统，建议使用 Windows 10 系统的 Linux 子系统来编译和运行。

<!-- more -->

Flink支持多种安装模式。

- local（本地）——单机模式，一般不使用 
- standalone——独立模式，Flink自带集群，开发测试环境使用 
- yarn——计算资源统一由Hadoop YARN管理，生产环境测试

# 2 基本概念

运行 Flink 应用其实非常简单，但是在运行 Flink 应用之前，还是有必要了解 Flink 运行时的各个组件，因为这涉及到 Flink 应用的配置问题。

Flink 实际运行时包括两类进程（下图所示）：

- JobManager（又称为 JobMaster）：协调 Task 的分布式执行，包括调度 Task、协调创建 Checkpoint 以及当 Job failover 时协调各个 Task 从 Checkpoint 恢复等。
- TaskManager（又称为 Worker）：执行 Dataflow 中的 Tasks，包括内存 Buffer 的分配、Data Stream 的传递等。

![Flink Runtime 架构图](https://gitee.com/littlefxc/oss/raw/master/images/3.png)

Flink Runtime 架构图说明：

- 当 Flink 集群启动后，首先会启动一个 JobManger 和一个或多个的 TaskManager。由 Client 提交任务给 JobManager，JobManager 再调度任务到各个 TaskManager 去执行，然后 TaskManager 将心跳和统计信息汇报给 JobManager。TaskManager 之间以流的形式进行数据的传输。上述三者均为独立的 JVM 进程。

- JobManager

  Master进程，负责Job的管理和资源的协调。包括任务调度，检查点管理，失败恢复等。

  当然，对于集群HA模式，可以同时多个master进程，其中一个作为leader，其他作为standby。当leader失败时，会选出一个standby的master作为新的leader（通过zookeeper实现leader选举）。


# 3 安装环境准备

- 从[官网](https://flink.apache.org/downloads.html)下载压缩包
- 安装Java，并配置 JAVA_HOME 环境变量

安装好Flink后，再来看下安装路径下的配置文件有哪些吧

![image-20210409111154176](https://gitee.com/littlefxc/oss/raw/master/images/image-20210409111154176.png)

安装目录下主要有 flink-conf.yaml 配置、日志的配置文件、zk 配置、Flink SQL Client 配置。

# 4 日志的查看和配置

JobManager 和 TaskManager 的启动日志可以在 Flink binary 目录下的 Log 子目录中找到。Log 目录中以`flink-{id}-${hostname}`为前缀的文件对应的是 JobManager 的输出，其中有三个文件：

- `flink-${user}-standalonesession-${id}-${hostname}.log`：代码中的日志输出
- `flink-${user}-standalonesession-${id}-${hostname}.out`：进程执行时的stdout输出
- `flink-${user}-standalonesession-${id}-${hostname}-gc.log`：JVM的GC的日志

Log 目录中以`flink-{id}-${hostname}`为前缀的文件对应的是 TaskManager 的输出，也包括三个文件，和 JobManager 的输出一致。

日志的配置文件在 Flink binary 目录的 conf 子目录下，其中：

- `log4j-cli.properties`：用 Flink 命令行时用的 log 配置，比如执行“ flink run”命令
- `log4j-yarn-session.properties`：用 `yarn-session.sh` 启动时命令行执行时用的 log 配置
- `log4j.properties`：无论是 Standalone 还是 Yarn 模式，JobManager 和 TaskManager 上用的 log 配置都是 log4j.properties

这三个`log4j.*properties`文件分别有三个`logback.*xml`文件与之对应，如果想使用 Logback 的同学，之需要把与之对应的`log4j.*properties`文件删掉即可，对应关系如下：

- `log4j-cli.properties -> logback-console.xml`
- `log4j-yarn-session.properties -> logback-yarn.xml`
- `log4j.properties -> logback.xml`

需要注意的是，“flink-{id}-和{user}-taskexecutor-{hostname}”都带有“，{id}”表示本进程在本机上该角色（JobManager 或 TaskManager）的所有进程中的启动顺序，默认从 0 开始。

# 5 配置文件 Flink-conf.yaml 详解

## 5.1 基础配置

```yaml
# jobManager 的IP地址
jobmanager.rpc.address: localhost

# JobManager 的端口号
jobmanager.rpc.port: 6123

# JobManager JVM heap 内存大小
jobmanager.heap.size: 1024m

# TaskManager JVM heap 内存大小
taskmanager.heap.size: 1024m

# 每个 TaskManager 提供的任务 slots 数量大小
taskmanager.numberOfTaskSlots: 1

# 程序默认并行计算的个数
parallelism.default: 1

# 文件系统来源, 使用本地文件系统："file:///", 使用 HDFS 分布式文件系统："hdfs://mynamenode:12345"
# fs.default-scheme
# fs.default-scheme: hdfs://localhost:9000
```

## 5.2 高可用

```yaml
# 可以选择 'NONE' 或者 'zookeeper'.
# high-availability: zookeeper

# 文件系统路径，让 Flink 在高可用性设置中持久保存元数据
# high-availability.storageDir: hdfs:///flink/ha/

# zookeeper 集群中仲裁者的机器 ip 和 port 端口号, host1:clientPort,host2:clientPort,...
# high-availability.zookeeper.quorum: localhost:2181

# 默认是 open，如果 zookeeper security 启用了该值会更改成 creator
# "creator" (ZOO_CREATE_ALL_ACL) 或者 "open" (ZOO_OPEN_ACL_UNSAFE)
# high-availability.zookeeper.client.acl: open
```

## 5.3 容错和检查点配置

```yaml
# 用于存储和检查点状态，可选值：'jobmanager', 'filesystem', 'rocksdb', or the <class-name-of-factory>
# state.backend: filesystem

# 存储检查点的数据文件和元数据的默认目录
# state.checkpoints.dir: hdfs://namenode-host:port/flink-checkpoints

# savepoints 的默认目标目录(可选)
# state.savepoints.dir: hdfs://namenode-host:port/flink-checkpoints

# 用于启用/禁用增量 checkpoints 的标志
# state.backend.incremental: false

# 故障转移策略，即作业计算如何从任务故障中恢复。
# 仅重新启动可能已受任务故障影响的任务，通常包括下游任务和潜在的上游任务（如果它们产生的数据不再可供使用）。
jobmanager.execution.failover-strategy: region
```

## 5.4 Restful 和 web 前端配置

```yaml
# 基于 Web 的运行时监视器侦听的地址.
#jobmanager.web.address: 0.0.0.0

# rest 端口，如果没有指定 rest.bind-port 的话会使用这个配置
rest.port: 8081

# rest 客户端地址
rest.address: 0.0.0.0

# rest 和 web 服务的端口范围
rest.bind-port: 8080-8090

# rest 和 web 服务的地址
rest.bind-address: 0.0.0.0

# 标记以指定是否从基于Web的运行时监视器启用作业提交。 取消注释以禁用。
# jobmanager.web.submit.enable: false

```

## 5.5 高级配置

```yaml
# io.tmp.dirs: /tmp

# 是否应在 TaskManager 启动时预先分配 TaskManager 管理的内存
# taskmanager.memory.preallocate: false

# 类加载解析顺序，是先检查用户代码 jar（“child-first”）还是应用程序类路径（“parent-first”）。 默认设置指示首先从用户代码 jar 加载类
# classloader.resolve-order: child-first


# 用于网络缓冲区的 JVM 内存的分数。 
# 这决定了 TaskManager 可以同时拥有多少流数据交换通道以及通道缓冲的程度。 
# 如果作业被拒绝或者您收到系统没有足够缓冲区的警告，请增加此值或下面的最小/最大值。 

# taskmanager.memory.network.fraction: 0.1
# taskmanager.memory.network.min: 64mb
# taskmanager.memory.network.max: 1gb
```

## 5.6 Flink 集群安全配置

```yaml
# 指示是否从 Kerberos ticket 缓存中读取
# security.kerberos.login.use-ticket-cache: true

# 包含用户凭据的 Kerberos 密钥表文件的绝对路径
# security.kerberos.login.keytab: /path/to/kerberos/keytab

# 与 keytab 关联的 Kerberos 主体名称
# security.kerberos.login.principal: flink-user

# 以逗号分隔的登录上下文列表，用于提供 Kerberos 凭据（例如，`Client，KafkaClient`使用凭证进行 ZooKeeper 身份验证和 Kafka 身份验证）
# security.kerberos.login.contexts: Client,KafkaClient
```

## 5.7 Zookeeper 安全配置

```yaml
# 覆盖以下配置以提供自定义 ZK 服务名称
# zookeeper.sasl.service-name: zookeeper

# 该配置必须匹配 "security.kerberos.login.contexts" 中的列表（含有一个）
# zookeeper.sasl.login-context-name: Client
```

## 5.8 HistoryServer

```yaml
# 你可以通过 bin/historyserver.sh (start|stop) 命令启动和关闭 HistoryServer

# 将已完成的作业上传到的目录
# jobmanager.archive.fs.dir: hdfs:///completed-jobs/

# 基于 Web 的 HistoryServer 的地址
# historyserver.web.address: 0.0.0.0

# 基于 Web 的 HistoryServer 的端口号
# historyserver.web.port: 8082

# 以逗号分隔的目录列表，用于监视已完成的作业
# historyserver.archive.fs.dir: hdfs:///completed-jobs/

# 刷新受监控目录的时间间隔（以毫秒为单位）
# historyserver.archive.fs.refresh-interval: 10000
```

# 6 masters

```
localhost:8081
```

# 7 workers

里面是每个 worker 节点的 IP/Hostname，每一个 worker 结点之后都会运行一个 TaskManager，一个一行。

```
localhost
```



# 8 zoo.cfg(可选)

Flink 自带的 zookeeper，如果使用外部独立的zookeeper

```
# 每个 tick 的毫秒数
tickTime=2000

# 初始同步阶段可以采用的 tick 数
initLimit=10

# 在发送请求和获取确认之间可以传递的 tick 数
syncLimit=5

# 存储快照的目录
# dataDir=/tmp/zookeeper

# 客户端将连接的端口
clientPort=2181

# ZooKeeper quorum peers
server.1=localhost:2888:3888
# server.2=host:peer-port:leader-port
```



# 参考资源

- [一文弄懂Flink基础理论](https://blog.csdn.net/oTengYue/article/details/102689538)
- [Akka在Flink中的应用](https://bigjar.github.io/2018/11/13/Akka在Flink中的应用/#Akka和Actor模型)
- [Akka and Actors](https://cwiki.apache.org/confluence/display/FLINK/Akka+and+Actors)
- [【Flink系列】之 Akka和Actors](https://blog.csdn.net/qq_19427611/article/details/80378273)

