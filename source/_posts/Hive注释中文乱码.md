---
title: Hive注释中文乱码
date: 2020-11-02 16:44:32
categories: 大数据
tags: 大数据,hive
---

[TOC]

# 1. Hive注释中文乱码

创建表的时候，comment说明字段包含中文，表成功创建成功之后，中文说明显示乱码

```sql
create table if not exists capacity_stats_live_access (
  id bigint comment '主键',
  device_num varchar(100) COMMENT '设备号',
  start_time timestamp COMMENT '调阅开始时间',
  finish_time timestamp COMMENT '调阅结束时间',
  duration int COMMENT '耗时，单位秒',
  response int COMMENT '调用结果 1.成功 0.失败'
) COMMENT '直播调阅日志统计';
```

![Hive%E6%B3%A8%E9%87%8A%E4%B8%AD%E6%96%87%E4%B9%B1%E7%A0%81%2044812d6495e341f4aec7117aac2ec503/Untitled.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/hive中文注释乱码.png)

这是因为在MySQL中的元数据出现乱码

# 2. 针对元数据库metastore中的表,分区,视图的编码设置

因为我们知道 metastore 支持数据库级别，表级别的字符集是 `latin1` 。

![Hive%E6%B3%A8%E9%87%8A%E4%B8%AD%E6%96%87%E4%B9%B1%E7%A0%81%2044812d6495e341f4aec7117aac2ec503/Untitled%201.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/Untitled%201-20201102170449594.png)

那么我们只需要把相应注释的地方的字符集由 latin1 改成 utf-8，就可以了。用到注释的就三个地方，表、分区、视图。如下修改分为两个步骤：

## 2.1. 进入数据库 Metastore 中执行以下 5 条 SQL 语句

修改表字段注解和表注解 `COLUMNS_V2`，`TABLE_PARAMS` 

修改分区字段注解 `PARTITION_PARAMS`，`PARTITION_KEYS` 

修改索引注解`INDEX_PARAMS` 

## 2.2. 修改 metastore 的连接 URL

修改hive-site.xml配置文件

```xml
<property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://IP:3306/db_name?createDatabaseIfNotExist=true&useUnicode=true&characterEncoding=UTF-8</value>
    <description>JDBC connect string for a JDBC metastore</description>
</property>
```

## 2.3. 验证

![Hive%E6%B3%A8%E9%87%8A%E4%B8%AD%E6%96%87%E4%B9%B1%E7%A0%81%2044812d6495e341f4aec7117aac2ec503/Untitled%202.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/Untitled%202-20201102170449708.png)

发现注释还是乱码。

把表删除然后重新创建。效果如下：

![Hive%E6%B3%A8%E9%87%8A%E4%B8%AD%E6%96%87%E4%B9%B1%E7%A0%81%2044812d6495e341f4aec7117aac2ec503/Untitled%203.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/Untitled%203-20201102170449863.png)