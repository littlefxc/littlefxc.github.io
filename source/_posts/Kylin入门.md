---
title: Kylin 入门
date: 2020-09-22 16:56:14
categories: 大数据
tags: kylin,大数据
---

[TOC]

# 介绍

随着移动互联网、物联网等技术的发展，近些年人类所积累的数据正在呈爆炸式的增长，大数据时代已经来临。但是海量数据的收集只是大数据技术的第一步，如何让数据产生价值才是大数据领域的终极目标。Hadoop的出现解决了数据存储问题，但如何对海量数据进行OLAP查询，却一直令人十分头疼。

企业中的查询大致可分为即席查询和定制查询两种。之前出现的很多OLAP引擎，包括Hive、Presto、SparkSQL等，虽然在很大程度上降低了数据分析的难度，但它们都只适用于即席查询的场景。它们的优点是查询灵活，但是随着数据量和计算复杂度的增长，响应时间不能得到保证。而定制查询多数情况下是对用户的操作做出实时反应，Hive等查询引擎动辄数分钟甚至数十分钟的响应时间显然是不能满足需求的。在很长一段时间里，企业只能对数据仓库中的数据进行提前计算，再将算好后的结果存储在MySQL等关系型数据库中，再提供给用户进行查询。但是当业务复杂度和数据量逐渐升高后，使用这套方案的开发成本和维护成本都显著上升。因此，如何对已经固化下来的查询进行亚秒级返回一直是企业应用中的一个痛点。

在这种情况下，Apache Kylin应运而生。不同于“大规模并行处理”（Massive Parallel Processing，MPP）架构的Hive、Presto等，Apache Kylin采用“预计算”的模式，用户只需要提前定义好查询维度，Kylin将帮助我们进行计算，并将结果存储到HBase中，为海量数据的查询和分析提供亚秒级返回，是一种典型的“空间换时间”的解决方案。Apache Kylin的出现不仅很好地解决了海量数据快速查询的问题，也避免了手动开发和维护提前计算程序带来的一系列麻烦。

Apache Kylin™是一个开源的、分布式的分析型数据仓库，提供 Hadoop 之上的 SQL 查询接口及多维分析（OLAP）能力以支持超大规模数据，最初由eBay Inc.开发并贡献至开源社区。

# Kylin 架构

如果想要知道Kylin是如何实现超大数据集的秒级多维分析查询，那么就得了解Kylin的架构原理。
Kylin实现秒级查询的关键点是预计算，对于超大数据集的复杂查询，既然现场计算需要花费较长时间，那么根据空间换时间的原理，我们就可以提前将所有可能的计算结果计算并存储下来，把高复杂度的聚合运算、多表连接等操作转换成对预计算结果的查询，比如把本该进行的Join、Sum、CountDistinct等操作改写成Cube的查询操作。从而实现超大数据集的秒级多维分析查询。

![image-20200924112848747](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20200924112848747.png)

- REST Server:
  REST Server是一套面向应用程序开发的入口点，旨在实现针对Kylin平台的应用开发工作。 此类应用程序可以提供查询、获取结果、触发cube构建任务、获取元数据以及获取用户权限等等。 另外可以通过Restful接口实现SQL查询。
- 查询引擎（Query Engine）:
  当cube准备就绪后，查询引擎就能够获取并解析用户查询。它随后会与系统中的其它组件进行交互，从而向用户返回对应的结果。
  在Kylin当中，我们使用一套名为Apache Calcite的开源动态数据管理框架对代码内的SQL以及其它插入内容进行解析。（Calcite最初被命名为Optiq，由Julian Hyde所编写，但如今已经成为Apache孵化器项目之一。）
- Routing
  负责将解析的SQL生成的执行计划转换成cube缓存的查询，cube是通过预计算缓存在hbase中，这部分查询可以在秒级设置毫秒级完成，而且还有一些操作使用过的查询原始数据（存储在Hadoop的hdfs中通过hive查询）。这部分查询延迟较高。
- 元数据管理工具（Metadata Manager）
  Kylin是一款元数据驱动型应用程序。元数据管理工具是一大关键性组件，用于对保存在Kylin当中的所有元数据进行管理，其中包括最为重要的cube元数据。其它全部组件的正常运作都需以元数据管理工具为基础。 Kylin的元数据存储在hbase中。
- 任务引擎（Cube Build Engine）:
  这套引擎的设计目的在于处理所有离线任务，其中包括shell脚本、Java API以及Map Reduce任务等等。任务引擎对Kylin当中的全部任务加以管理与协调，从而确保每一项任务都能得到切实执行并解决其间出现的故障。
- 存储引擎（Storage Engine）
  这套引擎负责管理底层存储——特别是cuboid，其以键-值对的形式进行保存。存储引擎使用的是HBase——这是目前Hadoop生态系统当中最理想的键-值系统使用方案。Kylin还能够通过扩展实现对其它键-值系统的支持，例如Redis。

**预计算大概流程**就是：将数据源中的数据按照指定的维度和指标，由计算引擎MapReduce离线计算出所有可能的查询结果(即Cube)存储到HBase中。HBase中每行记录的Rowkey由维度组成，度量会保存在 column family中。为了减小存储代价，这里会对维度和度量进行编码。查询阶段，利用HBase列存储的特性就可以保证Kylin有良好的快速响应和高并发。

需要注意的是：Kylin的三大依赖模块分别是数据源、构建引擎和存储引擎。默认这三者分别是Hive、MapReduce和HBase。但随着推广和使用的深入，渐渐有用户发现它们均存在不足之处。比如，实时分析可能会希望从Kafka导入数据而不是从Hive；而Spark的迅速崛起，又使我们不得不考虑将MapReduce替换为Spark，以期大幅提高Cube的构建速度；至于HBase，它的读性能可能还不如Cassandra或Kudu等。因此Kylin1.5版本的系统架构进行了重构，将数据源、构建引擎、存储引擎三大依赖抽象为接口，而Hive、MapReduce、HBase只是默认实现。深度用户可以根据自己的需要做二次开发，将其中的一个或多个替换为更适合的技术。

# 安装Kylin

## 单节点安装

```shell
ln -s /opt/apache-kylin-3.1.0-bin-hbase1x /usr/local/kylin
```

### 配置环境变量

```shell
vi /etc/profile
```

```bash
KYLIN_HOME=/usr/local/kylin
export KYLIN_HOME
PATH=$PATH:$JAVA_HOME/bin:$KYLIN_HOME/bin
```

```shell
source /etc/proifle
```

### macOS遇到的问题

```shell
bin/check-env.sh
```

此处mac有坑，因为检查脚本使用的是gnu的一些命令，和mac上的命令不同。所以需要安装gnu命令:

```shell
brew install gnu-sed
brew install findutils
```

### 启动

```shell
# 检查环境
/usr/local/kylin/bin/check-env.sh
# 启动
/usr/local/kylin/bin/kylin.sh start
# 样例
/usr/local/kylin/bin/sample.sh
# 重启
/usr/local/kylin/bin/kylin.sh restart
```

![image-20200924103518462](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20200924103518462.png)

# 核心概念

## 数据仓库

## OLAP

## 维度和度量

## Cube和Cuboid

## 事实表和维度表

## 星形模型
