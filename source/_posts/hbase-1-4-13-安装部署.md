---
title: hbase 1.4.13 安装部署
date: 2020-09-16 17:19:13
categories: 大数据
tags: hbase,大数据
---

[TOC]

# 认识HBase

## HBase介绍

HBase = Hadoop database，Hadoop数据库
开源数据库
官网：[hbase.apache.org/](http://hbase.apache.org/)
HBase源于Google的BigTable
Apache HBase™是Hadoop数据库，是一个分布式，可扩展的大数据存储。
当需要对大数据进行随机、实时读/写访问时，请使用Apache HBase™。该项目的目标是托管非常大的表 - 数十亿行X百万列 - 在商品硬件集群上。Apache HBase是一个开源的，分布式的，版本化的非关系数据库nosql，模仿Google的Bigtable： Chang等人的结构化数据分布式存储系统。正如Bigtable利用Google文件系统提供的分布式数据存储一样，Apache HBase在Hadoop和HDFS之上提供类似Bigtable的功能。
HBase可执行基于Yarn平台的计算任务，但不擅长。

< !--more-->

## HBase集群角色

- HDFS：
  - NameNode——主节点
  - DataNode——数据存储节点
- Yarn：
  - ResourceManager——全局的资源管理器
  - NodeManager——分节点资源和任务管理器
- HBase：
  - HMaster
    - 负责Table表和RegionServer的监控管理工作
    - 处理元数据的变更
    - 对HRegionServer进行故障转移
    - 空闲时对数据进行负载均衡处理
    - 管理Region
    - 借助ZooKeeper发布位置到客户端
  - HRegionServer
    - 负责Table数据的实际读写
    - 刷新缓存数据到HDFS
    - 处理Region
    - 可以进行数据压缩
    - 维护Hlog
    - Region分片

## hbase 架构

![image-20200916184659443](https://gitee.com/littlefxc/oss/raw/master/images/image-20200916184659443.png)

HRegionServer结构：

- HLog：存储HBase的修改记录
- HRegion：根据rowkey（行键，类似id）分割的表的分片
- Store：对应HBase表中的一个列族，可存储多个字段
- HFile：真正的存储文件
- MemStore：保存当前的操作
- ZooKeeper：存放数据的元数据信息，负责维护RegionServer中保存的元数据信息
- DFS Client：存储数据信息到HDFS集群中

# 安装

## 前期准备

- Hadoop集群环境
- ZooKeeper集群环境

## 服务器分配

| 主机名       | IP             |
| ------------ | -------------- |
| centos-node1 | 192.168.99.101 |
| centos-node2 | 192.168.99.102 |
| centos-node3 | 192.168.99.103 |

## 解压安装文件到指定目录

```sh
$ tar -zxvf hbase-1.4.13.tar.gz -C 目标目录
```

## 修改配置文件

### hbase-env.sh

```sh
$ vi hbase-env.sh
```

```bash
 # The java implementation to use.  Java 1.7+ required.
 export JAVA_HOME=jdk安装路径

 # 注释掉以下语句（jdk1.8中不需要这个配置）
 # export HBASE_MASTER_OPTS="$HBASE_MASTER_OPTS -XX:PermSize=128m -XX:MaxPermSize=128m"
 # export HBASE_REGIONSERVER_OPTS="$HBASE_REGIONSERVER_OPTS -XX:PermSize=128m -XX:MaxPermSize=128m"

 # Tell HBase whether it should manage it's own instance of Zookeeper or not.
 # 关闭HBase自带的ZooKeeper
 export HBASE_MANAGES_ZK=false
```

### hbase-site.xml

```sh
$ vi hbase-site.xml
```

```xml
<configuration>
	<!-- 设置namenode所在位置（HDFS中存放的路径） -->
	<property>
	    <name>hbase.rootdir</name>
	    <value>hdfs://centos-node1:9000/hbase</value>    
	</property>
	
	<!-- 是否开启集群 -->
	<property>
	    <name>hbase.cluster.distributed</name>
	    <value>true</value>
	</property>
  
	<!-- HBase-0.9.8之前默认端口为60000 -->
	<property>
	    <name>hbase.master.port</name>
	    <value>16000</value>
	</property>
  
  <property>
		<name>hbase.master.info.port</name>
		<value>16010</value>
	</property>
  
  <property>
		<name>hbase.regionserver.port</name>
		<value>16201</value>
	</property>
  
  <property>
		<name>hbase.regionserver.info.port</name>
		<value>16301</value>
	</property>
	
	<!-- zookeeper集群的位置 -->
	<property>
	    <name>hbase.zookeeper.quorum</name>
	    <!-- 注意不要有空格 -->  
	    <value>centos-node1,centos-node2,centos-node3</value>
	</property>
	
	<!-- hbase的元数据信息存储在zookeeper的位置 -->
	<property>
	    <name>hbase.zookeeper.property.dataDir</name>
	    <value>/usr/local/zookeeper/data</value>
	</property>
</configuration>
```

### regionservers

加入节点主机名

```bash
centos-node1
centos-node2
centos-node3
```

## 解决依赖问题

```sh
$ cd /usr/local/hbase/lib
$ rm -fr hadoop-*
$ rm -fr zookeeper-*
```

把相关版本的zookeeper和hadoop的依赖包导入到hbase/lib

## 软链接hadoop配置

```sh
$ ln -s /usr/local/hadoop/etc/hadoop/core-site.xml /usr/local/hbase/conf/core-site.xml
$ ln -s /usr/local/hadoop/etc/hadoop/hdfs-site.xml /usr/local/hbase/conf/hdfs-site.xml
```

## 复制到其他节点

```sh
$ scp -r /usr/local/hbase centos-node2:/usr/local/
$ scp -r /usr/local/hbase centos-node3:/usr/local/
```

## 配置环境变量

```sh
$ vi /etc/profile
```

```bash
JAVA_HOME=/usr/local/java/
JRE_HOME=$JAVA_HOME/jre
HADOOP_HOME=/usr/local/hadoop
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
ZOOKEEPER_HOME=/usr/local/zookeeper
HBASE_HOME=/usr/local/hbase

PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin:$HADOOP_HOME/bin:$ZOOKEEPER_HOME/bin:$HBASE_HOME/bin

export JAVA_HOME JRE_HOME PATH CLASSPATH HADOOP_HOME ZOOKEEPER_HOME HBASE_HOME
```

```sh
$ source /etc/profile
```

## 修改防火墙

```sh
# 查看当前区域
$ firewall-cmd --get-active-zones
# 新建一个自定义服务(端口不全)
$ firewall-cmd --new-service=hbase --permanent
$ firewall-cmd --service=hbase --add-port 16000/tcp --permanent
$ firewall-cmd --service=hbase --add-port 16010/tcp --permanent
$ firewall-cmd --service=hbase --add-port 16201/tcp --permanent
$ firewall-cmd --service=hbase --add-port 16301/tcp --permanent
$ firewall-cmd --service=hbase --add-port 2181/tcp --permanent
# 不中断服务的重新加载
$ firewall-cmd --reload
$ firewall-cmd --add-service=hbase
# 将当前防火墙的规则永久保存；
$ firewall-cmd --runtime-to-permanent
```

或者

```sh
$ systemctl stop firewalld
$ systemctl disable firewalld
```



# 启动集群

- 启动主节点
  - `hbase-daemon.sh start master`
- 启动从节点
  - `hbase-daemon.sh start regionserver`

# 关闭集群

- 关闭主节点
  - `hbase-daemon.sh stop master`
- 关闭从节点
  - `hbase-daemon.sh stop regionserver`

# UI 展示

![image-20200921184311647](https://gitee.com/littlefxc/oss/raw/master/images/image-20200921184311647.png)

# hbase shell的一些命令

```sh
# 进入终端
$ hbase shell
```

![image-20200921185108081](https://gitee.com/littlefxc/oss/raw/master/images/image-20200921185108081.png)

```sh
# 查询表
$ list
```

![image-20200921185132638](https://gitee.com/littlefxc/oss/raw/master/images/image-20200921185132638.png)

```sh
# 显示服务器状态
$ status
```

![image-20200921185249611](https://gitee.com/littlefxc/oss/raw/master/images/image-20200921185249611.png)

```sh
# 显示当前用户
$ whoami
```

![image-20200921185450703](https://gitee.com/littlefxc/oss/raw/master/images/image-20200921185450703.png)

```sh
# 创建表
$ create '表名', '列族1', '列族2'
```

![image-20200921185551041](https://gitee.com/littlefxc/oss/raw/master/images/image-20200921185551041.png)

```sh
# 表的一些操作
# 全表扫描
$ scan '表名'
# 指定Rowkey扫描
$ scan '表名', {STARTROW => 'Rowkey值', STOPROW => 'Rowkey值'}
# 查看表结构
$ describe '表名'
# 修改表结构信息
$ alter '表名', {NAME => '列族名', 变更字段名 => ' '}
# 查询指定数据信息
# 指定具体的rowkey
$ get '表名', 'rowkey'
# 指定具体的列
$ get '表名', 'rowkey', '列族:列名'
# 统计表行数
$ count '表名'
# 根据Rowkey进行统计
$ count '表名', 'rowkey'
# 表中添加数据信息。HBase只有覆盖没有修改,覆盖时对应表名、rowkey、列族、列名字段，输入新的值信息
$ put '表名', 'rowkey', '列族:列名', '值'
# 清空表
$ truncate '表名'
# 删除表
# 指定表不可用
$ disable '表名'
# 删除
$ drop '表名'
# 退出终端
$ exit
$ quit
```

# 参考资源

- https://juejin.im/post/6844903797655863309#heading-21

