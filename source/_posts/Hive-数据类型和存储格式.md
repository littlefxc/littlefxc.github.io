---
title: Hive 数据类型和存储格式
date: 2020-09-24 19:02:20
categories: 大数据
tags: 大数据,hive
---

[【Hive教程】（二）Hive数据类型和存储格式](http://bigdata-star.com/archives/1013)

[Apache Software Foundation](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Types)

[Hive复合数据类型array,map,struct的使用_Life is for sharing的博客-CSDN博客](https://blog.csdn.net/sl1992/article/details/53894481?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.channel_param)

# **HIVE数据类型**

毕竟HIVE穿着SQL的外壳，肯定支持诸如Mysql这种RDBMS的数据类型，如`int`,`varchar`,但是它还具有非常多自有的数据类型，包括复杂的数据类型（数组，Map等）也是支持的！

![Hive 支持的数据类型 - 摘自官网wiki](https://gitee.com/littlefxc/oss/raw/master/images/Untitled.png)

数字类型，日期类型，String类型，Boolean类型我们都是比较熟悉的，也比较简单，就不讲解了。演示一下复杂数据类型：

## arrays

```sql
hive (hive)> create table arraytest (id int,course array<string>)
           > row format delimited fields terminated by','
           > collection items terminated by':';
```

说明：

- `row format delimited fields terminated by','` 是指定列与列之间的分隔符,此处为”,”
- `collection items terminated by':'` 是指定集合内元素之间的分隔符,此处为”：”

因此我们要导入到hive中的数据应该是形如：

```
1,math:chinese
2,english:history
```

数据加载到数据库

```sql
hive> load data local inpath '/Users/fengxuechao/WorkSpace/software/hive_data/arraytest.txt' into table arraytest;
Loading data to table default.arraytest
OK
Time taken: 0.963 seconds
```

查询所有数据：

```sql
hive> select * from arraytest;
OK
1	["math","chinese"]
2	["english","history"]
Time taken: 1.297 seconds, Fetched: 2 row(s)
```

查询数组指定索引的所有数据：

```sql
hive> select course[1] from arraytest;
OK
chinese
history
Time taken: 0.364 seconds, Fetched: 2 row(s)
```

## maps

创建表

```sql
hive> create table maptest(name string,score map<string,float>)
    > row format delimited fields terminated by','
    > collection items terminated by '|'
    > map keys terminated by':';
OK
Time taken: 0.046 seconds
```

数据

```sql
'小明',math:96|chinese:95
'小红',math:80|chinese:99
```

数据加载到数据库

```sql
hive> load data local inpath '/Users/fengxuechao/WorkSpace/software/hive_data/maptest.txt' into table maptest;
Loading data to table default.maptest
OK
Time taken: 0.293 seconds
```

查询所有数据：

```sql
hive> select * from maptest;
OK
'小明'	{"math":96.0,"chinese":95.5}
'小红'	{"math":80.0,"chinese":99.0}
Time taken: 0.1 seconds, Fetched: 2 row(s)
```

查询数组指定key的所有数据：

```sql
hive> select name,score['math'] from maptest;
OK
'小明'	96.0
'小红'	80.0
Time taken: 0.112 seconds, Fetched: 2 row(s)
```

## structs

创建表

```sql
hive> create table struct_test(name string,course struct<course:string,score:int>)
    > row format delimited fields terminated by ','
    > collection items terminated by ':';
OK
Time taken: 0.047 seconds
```

数据

```sql
小明,math:79
小红,math:80
```

数据加载到数据库

```sql
hive> load data local inpath '/Users/fengxuechao/WorkSpace/software/hive_data/struct_test.txt' into table struct_test;
Loading data to table default.struct_test
OK
Time taken: 0.293 seconds
```

查询所有数据：

```sql
hive> select * from struct_test;
OK
小明	{"course":"math","score":79}
小红	{"course":"math","score":80}
Time taken: 0.097 seconds, Fetched: 2 row(s)
hive> select name,course.course,course.score from struct_test;
OK
小明	math	79
小红	math	80
Time taken: 0.213 seconds, Fetched: 2 row(s)
```
