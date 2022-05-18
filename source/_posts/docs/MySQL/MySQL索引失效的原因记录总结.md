---
title: "MySQL索引失效的原因记录总结"
date: 2022-03-22T14:55:56+08:00
categories: 数据库
tags:
- mysql
---

## 0 引言

好记性不如烂笔头，把常见的一些 MySQL 索引失效的问题记录下来，在工作中可以时时检查对比。

主要分为两个部分，explain 介绍和各种索引失效场景的模拟。

### 建表语句

```sql
CREATE TABLE `people` (
  `id` int unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(30) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL COMMENT '姓名',
  `gender` tinyint unsigned DEFAULT NULL COMMENT '性别，0男1女',
  `career` varchar(30) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL COMMENT '技能',
  `skills` varchar(30) COLLATE utf8mb4_general_ci DEFAULT NULL COMMENT '技能',
  `birthday` date DEFAULT NULL COMMENT '出生日期',
  `lifetime` int DEFAULT NULL COMMENT '寿命',
  `gmt_create` timestamp NULL DEFAULT NULL COMMENT '入库时间',
  PRIMARY KEY (`id`),
  KEY `idx_career_skills_lifetime` (`career`,`skills`,`lifetime`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=26 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

### `explain` 执行计划各属性解释说明

#### id

SELECT 识别符。这是 SELECT 的查询序列号，SQL 的执行顺序：

1. id 相同时，执行顺序自上而下。
2. 如果是子查询，id 会递增，id 值越大优先级越高，越先被执行。
3. id 如果相同，可以认为是一组，自上而下顺序执行；再所有组中，优先级越高，越先被执行。

#### select_type

| select_type        | 说明                                                         |
| ------------------ | ------------------------------------------------------------ |
| SIMPLE             | 简单查询，不使用UNION或子查询                                |
| PRIMARY            | 最外层查询，查询中若包含任何复杂的子部分，最外层的 select 被标记为 PRIMARY |
| SUBQUERY           | 子查询中的第一个select，结果不依赖于外部查询                 |
| DEPENDENT SUBQUERY | 子查询中的第一个select，结果依赖于外部查询                   |
| UNCACHEABLE SUBQUERY | 一个子查询的结果不能被缓存，必需重新评估外链表的第一行        |
| DERIVED            | 子查询，派生表的 select，from 子句的的子查询                 |
| UNION              | 联合，UNION 中的第二个或后面的SELECT语句                          |
| UNION RESULT       | 使用联合的结果                                               |
| DEPENDENT UNION | UNION中的第二个或后面的SELECT语句，取决于外面的查询 |

#### type

| type        | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| ALL         | 全数据表扫描                                                 |
| index       | 全索引表扫描                                                 |
| RANGE       | 对索引列进行范围查询                                         |
| INDEX_MERGE | 合并索引，使用多个单列索引查询                               |
| REF         | 根据索引查找一个或多个值                                     |
| EQ_REF      | 搜索时使用 primary key 或 unique 类型                        |
| CONST       | 常量，表最多有一个匹配行,因为仅有一行,在这行的列值可被优化器剩余部分认为是常数,const表很快,因为它们只读取一次。 |
| SYSTEM      | 系统，表仅有一行(=系统表)。这是const联接类型的一个特例。     |

- 性能：all  <  index  <  range  <  index_merge  <  ref_or_null  <  ref  <  eq_ref  <  system/const

- 性能在 range 之下基本都可以进行调优

#### possible_keys

可能使用的索引

#### key

真实使用的索引

#### key_len

MySQL中使用索引字节长度

#### rows

mysql 预估为了找到所需的行而要读取的行数

#### filtered

按表条件过滤的行百分比

#### extra

| extra                                       | 说明                                                         |
| :------------------------------------------ | :----------------------------------------------------------- |
| Using index                                 | 此值表示mysql将使用覆盖索引，以避免访问表。                  |
| Using where                                 | mysql 将在存储引擎检索行后再进行过滤，许多where条件里涉及索引中的列，当(并且如果)它读取索引时，就能被存储引擎检验，因此不是所有带where子句的查询都会显示“Using where”。有时“Using where”的出现就是一个暗示：查询可受益于不同的索引。 |
| Using temporary                             | mysql 对查询结果排序时会使用临时表。常见于排序和分组查询`group by`,`order by` |
| Using filesort                              | 当Query中包含 order by 操作，而且无法利用索引完成的排序操作称为“文件排序”。 |
| Range checked for each record(index map: N) | 没有好用的索引，新的索引将在联接的每一行上重新估算，N是显示在possible_keys列中索引的位图，并且是冗余的 |
| ...                                         | ...                                                          |

## 1 不符合最左前缀原则

```sql
mysql> explain select * from people where `lifetime` = 23 and `skills` = '口才';
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | people | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   14 |     7.14 | Using where |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

说明：

- 最左前缀原则指的是从索引最左前列开始并且不跳过索引中的列

- type 是 ALL, 表示查询语句是全表数据查询，where 的查询条件是 lifetime 和 skills，缺少 career 这个索引条件，无法命中索引 idx_career_skills_lifetime。

## 2 在索引列上有多余操作，如：函数、计算、类型转换

```sql
mysql> explain select * from people where left(career, 2) = '群众';
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | people | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   14 |   100.00 | Using where |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------------+
```

说明：

- `LEFT()`函数是一个字符串函数，它返回具有指定长度的字符串的左边部分。
- `LEFT()`函数的语法：`LEFT(str,length);`
  - `str`是要提取子字符串的字符串。
  - `length`是一个正整数，指定将从左边返回的字符数。
  - `LEFT()`函数返回`str`字符串中最左边的长度字符。
  - 如果`str`或`length`参数为`NULL`，则返回`NULL`值。
  - 如果`length`为`0`或为负，则`LEFT`函数返回一个空字符串。
  - 如果`length`大于`str`字符串的长度，则`LEFT`函数返回整个`str`字符串。
  - 请注意，`SUBSTRING` 或 `SUBSTR` 函数也提供与`LEFT`函数相同的功能。
- career 字段是有索引的，但是执行计划中看到的却没有命中，说明在索引字段上添加多余操作会使其失效


## 3 查询条件有`!=`、`>`	、`<`等

## 4 `like`以通配符`%`为开头

## 5 使用来了`or`

## 6 使用了`is null`或者`is not null`
