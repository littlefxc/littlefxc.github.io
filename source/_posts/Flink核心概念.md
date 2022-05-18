---
title: Flink核心概念
date: 2021-04-08 19:55:12
categories: 大数据
tags:
- flink
---

# 前言

运行 Flink 应用其实非常简单，但是在运行 Flink 应用之前，还是有必要了解 Flink 运行时的各个组件，因为这涉及到 Flink 应用的配置问题。下图 所示，这是用户用 DataStream API 写的一个数据处理程序。可以看到，在一个 DAG 图中不能被 Chain 在一起的 Operator 会被分隔到不同的 Task 中，也就是说 Task 是 Flink 中资源调度的最小单位。

<!-- more -->

![图2](https://gitee.com/littlefxc/oss/raw/master/images/2-20210408195606994.png)

Flink 实际运行时包括两类进程（下图所示）：

- JobManager（又称为 JobMaster）：协调 Task 的分布式执行，包括调度 Task、协调创建 Checkpoint 以及当 Job failover 时协调各个 Task 从 Checkpoint 恢复等。
- TaskManager（又称为 Worker）：执行 Dataflow 中的 Tasks，包括内存 Buffer 的分配、Data Stream 的传递等。

![Flink Runtime 架构图](https://gitee.com/littlefxc/oss/raw/master/images/3-20210408195607193.png)

Flink Runtime 架构图说明：

- 当 Flink 集群启动后，首先会启动一个 JobManger 和一个或多个的 TaskManager。由 Client 提交任务给 JobManager，JobManager 再调度任务到各个 TaskManager 去执行，然后 TaskManager 将心跳和统计信息汇报给 JobManager。TaskManager 之间以流的形式进行数据的传输。上述三者均为独立的 JVM 进程。

- JobManager

  Master进程，负责Job的管理和资源的协调。包括任务调度，检查点管理，失败恢复等。

  当然，对于集群HA模式，可以同时多个master进程，其中一个作为leader，其他作为standby。当leader失败时，会选出一个standby的master作为新的leader（通过zookeeper实现leader选举）。


从下图中可以看出 Task Slot 是一个 TaskManager 中的最小资源分配单位，一个 TaskManager 中有多少个 Task Slot 就意味着能支持多少并发的 Task 处理。需要注意的是，一个 Task Slot 中可以执行多个 Operator，一般这些 Operator 是能被 Chain 在一起处理的。

![Process](https://gitee.com/littlefxc/oss/raw/master/images/4-20210408195607317.png)

# 1 Apache Flink 的定义、架构及原理

Apache Flink 是一个分布式大数据处理引擎，可对有限数据流和无限数据流进行有状态或无状态的计算，能够部署在各种集群环境，对各种规模大小的数据进行快速计算。

了解Flink 应用开发需要先理解Flink 的Streams、State、Time 等基础处理语义以及Flink 兼顾灵活性和方便性的多层次API。

- **Streams**：流，分为有限数据流与无限数据流，unbounded stream 是有始无终的数据流，即无限数据流；而bounded stream 是限定大小的有始有终的数据集合，即有限数据流，二者的区别在于无限数据流的数据会随时间的推演而持续增加，计算持续进行且不存在结束的状态，相对的有限数据流数据大小固定，计算最终会完成并处于结束的状态。
- **State**，状态是计算过程中的数据信息，在容错恢复和Checkpoint 中有重要的作用，流计算在本质上是Incremental Processing，因此需要不断查询保持状态；另外，为了确保Exactly- once 语义，需要数据能够写入到状态中；而持久化存储，能够保证在整个分布式系统运行失败或者挂掉的情况下做到Exactly- once，这是状态的另外一个价值。
- **Time**，分为Event time、Ingestion time、Processing time，Flink 的无限数据流是一个持续的过程，时间是我们判断业务状态是否滞后，数据处理是否及时的重要依据。
- **API**，API 通常分为三层，由上而下可分为SQL / Table API、DataStream API、ProcessFunction 三层，API 的表达能力及业务抽象能力都非常强大，但越接近SQL 层，表达能力会逐步减弱，抽象能力会增强，反之，ProcessFunction 层API 的表达能力非常强，可以进行多种灵活方便的操作，但抽象能力也相对越小。

# 2 Flink 架构

在架构部分，主要分为以下四点：

第一，Flink 具备统一的框架处理有界和无界两种数据流的能力

第二， 部署灵活，Flink 底层支持多种资源调度器，包括Yarn、Kubernetes 等。Flink 自身带的Standalone 的调度器，在部署上也十分灵活。

第三， 极高的可伸缩性，可伸缩性对于分布式系统十分重要，阿里巴巴双11大屏采用Flink 处理海量数据，使用过程中测得Flink 峰值可达17 亿/秒。

第四， 极致的流式处理性能。Flink 相对于Storm 最大的特点是将状态语义完全抽象到框架中，支持本地状态读取，避免了大量网络IO，可以极大提升状态存取的性能。

# 3 Flink Operation

后面会专门讲解，此处简单分享Flink 关于运维及业务监控的内容：

- Flink具备7 X 24 小时高可用的SOA（面向服务架构），原因是在实现上Flink 提供了一致性的Checkpoint。Checkpoint是Flink 实现容错机制的核心，它周期性的记录计算过程中Operator 的状态，并生成快照持久化存储。当Flink 作业发生故障崩溃时，可以有选择的从Checkpoint 中恢复，保证了计算的一致性。
- Flink本身提供监控、运维等功能或接口，并有内置的WebUI，对运行的作业提供DAG 图以及各种Metric 等，协助用户管理作业状态。

# 4 Flink 的应用场景

##  4.1 Flink的应用场景：Data Pipeline

![img](https://gitee.com/littlefxc/oss/raw/master/images/2-16.004.jpeg)

Data Pipeline 的核心场景类似于数据搬运并在搬运的过程中进行部分数据清洗或者处理，而整个业务架构图的左边是Periodic ETL，它提供了流式ETL 或者实时ETL，能够订阅消息队列的消息并进行处理，清洗完成后实时写入到下游的Database或File system 中。场景举例：

###  实时数仓

当下游要构建实时数仓时，上游则可能需要实时的Stream ETL。这个过程会进行实时清洗或扩展数据，清洗完成后写入到下游的实时数仓的整个链路中，可保证数据查询的时效性，形成实时数据采集、实时数据处理以及下游的实时Query。

### 搜索引擎推荐

搜索引擎这块以淘宝为例，当卖家上线新商品时，后台会实时产生消息流，该消息流经过Flink 系统时会进行数据的处理、扩展。然后将处理及扩展后的数据生成实时索引，写入到搜索引擎中。这样当淘宝卖家上线新商品时，能在秒级或者分钟级实现搜索引擎的搜索。

## 4.2 Flin 应用场景：Data Analytics

![img](https://gitee.com/littlefxc/oss/raw/master/images/3-.005.jpeg)

Data Analytics，如图，左边是Batch Analytics，右边是Streaming Analytics。Batch Analytics 就是传统意义上使用类似于Map Reduce、Hive、Spark Batch 等，对作业进行分析、处理、生成离线报表；Streaming Analytics 使用流式分析引擎如Storm、Flink 实时处理分析数据，应用较多的场景如实时大屏、实时报表。

## 4.3 Flink 应用场景：Data Driven

![img](https://gitee.com/littlefxc/oss/raw/master/images/4-.006.jpeg)

从某种程度上来说，所有的实时的数据处理或者是流式数据处理都是属于Data Driven，流计算本质上是Data Driven 计算。应用较多的如风控系统，当风控系统需要处理各种各样复杂的规则时，Data Driven 就会把处理的规则和逻辑写入到Datastream 的API 或者是ProcessFunction 的API 中，然后将逻辑抽象到整个Flink 引擎，当外面的数据流或者是事件进入就会触发相应的规则，这就是Data Driven 的原理。在触发某些规则后，Data Driven 会进行处理或者是进行预警，这些预警会发到下游产生业务通知，这是Data Driven 的应用场景，Data Driven 在应用上更多应用于复杂事件的处理。

# 参考资料

[Apache Flink 零基础入门（一&二）：基础概念解析](https://ververica.cn/developers/flink-basic-tutorial-1-basic-concept/)