---
title: Hive集成HBase 详解
date: 2020-11-11 16:31:29
tags:
---

[TOC]

# 1 前言

## 1.1 为什么要集成Hive和HBase?

Kylin的三大依赖模块分别是数据源、构建引擎和存储引擎。默认这三者分别是Hive、MapReduce和HBase。但随着调研和使用的深入，渐渐有用户发现它们均存在不足之处。

比如 Hive 不应该用来进行实时的查询，因为它需要很长时间才可以返回结果。同时想要将数据实时的插入hive中则完全没有可操作性。

至于 HBase虽然非常适用于海量明细数据（十亿、百亿）的随机实时查询但只提供了简单的基于 Key 值的快速查询能力—，没法进行大量的条件查询，对于数据分析来说，不太友好。

在大数据架构中，Hive和HBase是协作关系，Hive方便地提供了Hive QL的接口来简化MapReduce的使用， 而HBase提供了低延迟的数据库访问。如果两者结合，可以利用MapReduce的优势针对HBase存储的大量内容进行离线的计算和分析。

# 2 Hive集成HBase的原理

Hive 整合 hbase 为用户提供一种 sqlOnHbase 的方法。Hive 与 HBase 整合的实现是利用两者本身对外的 API 接口互相通信来完成的，其具体工作交由 Hive 的 lib 目录中的 `hive-hbase-handler-xxx.jar` 工具类来实现对 HBase 数据的读取。

通过HBaseStorageHandler，Hive可以获取到Hive表所对应的HBase表名，列簇和列，InputFormat、OutputFormat类，创建和删除HBase表等。

Hive访问HBase中HTable的数据，实质上是通过MR读取HBase的数据，而MR是使用HiveHBaseTableInputFormat完成对表的切分，获取RecordReader对象来读取数据的。

对HBase表的切分原则是一个Region切分成一个Split,即表中有多少个Regions,MR中就有多少个Map；

读取HBase表数据都是通过构建Scanner，对表进行全表扫描，如果有过滤条件，则转化为Filter。当过滤条件为rowkey时，则转化为对rowkey的过滤；Scanner通过RPC调用RegionServer的next()来获取数据。

# 3 使用场景

# 4 整合

因为Hive与HBase集成是利用两者本身对外的API接口互相通信来完成的，其具体工作交由Hive的lib目录中的hive-hbase-handler-.jar工具类来实现。所以只需要将hive的 `hive-hbase-handler-.jar` 复制到`hbase/lib`中就可以了。

```
cp /usr/local/hive/lib/hive-hbase-handler-2.3.7.jar /usr/local/hbase/lib/
```

**注**: 如果在hive整合hbase中，出现版本之类的问题，那么以hbase的版本为主，将hbase中的jar包覆盖hive的jar包。

## 4.1 示例

```sql
# Hive 语句
create table t_employee(id int,name string) 
stored by 'org.apache.hadoop.hive.hbase.HBaseStorageHandler' 
with serdeproperties("hbase.columns.mapping"=":key,st1:name") 
tblproperties("hbase.table.name"="t_employee","hbase.mapred.output.outputtable" = "t_employee");

# hbase 语句
list
describe 't_employee'
# 返回结果
Table t_employee is ENABLED
t_employee
COLUMN FAMILIES DESCRIPTION
{NAME => 'st1', BLOOMFILTER => 'ROW', VERSIONS => '1', IN_MEMORY => 'false', KEEP_DELETED_CELLS => 'FALSE', DATA_BLOCK_ENCODING => 'NONE', TTL => 'FOREVER', COMPRESSION => 'NONE'
, MIN_VERSIONS => '0', BLOCKCACHE => 'true', BLOCKSIZE => '65536', REPLICATION_SCOPE => '0'}
1 row(s) in 0.0260 seconds
```

说明：

1. 这里面出现了三个t_employee表名，第一个t_employee 是hive表中的名称，(id int,name string) 是hive表结构。

   在tblproperties 语句中还出现了2个t_employee表，"hbase.table.name"定义的是在hbase中的表名 ，这个属性是可选的，仅当你想在Hive和Hbase中使用不同名字的表名时才需要填写，如果使用相同的名字则可以省略；

   "hbase.mapred.output.outputtable"定义的第三个t_employee是存储数据表的名称，指定插入数据时写入的表，如果以后需要往该表插入数据就需要指定该值，这个可以不要，表数据就存储在第二个表中了 。

2. stored by 'org.apache.hadoop.hive.hbase.HBaseStorageHandler' ：是指定处理的存储器，就是hive-hbase-handler-*.jar包，要做hive和hbase的集成必须要加上这一句；

3. “hbase.columns.mapping” 是定义在hive表中的字段怎么与hbase的列族进行映射。

      例如:st1就是列族，name就是列。它们之间通过“：”连接。

      在hive中创建的t_employee表，包括两个字段（int型的id和string型的name），映射为hbase中的表t_employee，其中：key对应hbase的rowkey，value对应hbase的st1:name列。

## 4.2 字段映射

控制HBase字段和Hive之间的映射有两种`SERDEPROPERTIES`:

- hbase.columns.mapping
- hbase.table.default.storage.type，可以是string(default)或binary中的任一个，指定这个选项只有在Hive 0.9之后可使用.

目前所支持的字段映射多少是有些难处理或存在约束的：

- 对于每一个Hive字段，表的创建者必须用逗号分隔的字符串（`hbase.columns.mapping`）指定对应的入口（Hive表有n个字段，则该字符串得指定n个入口），在各个入口之间不能由空格（因为空格会被解析成字段名中的一部分）。
- 映射入口必须是以下两者之一：**行健**或**'列族名:[列名][#(binary|string)]'**
  - 如果没有指定类型，则直接使用`hbase.table.default.storage.type`的值
  - 合法值的的前缀也是合法的（例如#b表示#binary）
  - 如果指定某字段为binary，则对应的HBase中的单元格则应该是HBase的Bytes类的内容组成
- 必须要有确切的行健映射
- 如果没有指定列名，则默认使用Hive的字段名作为HBase中的列名

## 4.3 数据同步测试

进入hbase之后，在t_employee中添加两条数据 然后查询该表

```sql
put 't_employee','1001','st1:name','zhaoqian'
put 't_employee','1002','st1:name','sunli'
scan 't_employee'
```

查询结果：

![Hive%E9%9B%86%E6%88%90HBase%20%E8%AF%A6%E8%A7%A3%20100037f2aa264a2c835a0530a373a912/Untitled.png](https://gitee.com/littlefxc/oss/raw/master/images/hive_on_hbase.png)

然后切换到hive中查询该表

![Hive%E9%9B%86%E6%88%90HBase%20%E8%AF%A6%E8%A7%A3%20100037f2aa264a2c835a0530a373a912/Untitled%201.png](https://gitee.com/littlefxc/oss/raw/master/images/hive_on_hbase2.png)

至此，hive 集成hbase 就到此告一段落。

# 参考资源

[如何整合hive和hbase](https://zhuanlan.zhihu.com/p/74041611)

[Hive与HBase的集成实践](https://ask.hellobi.com/blog/marsj/4002)

[[hbase-1-4-13-安装部署]]

[[hive2-3-7安装]]