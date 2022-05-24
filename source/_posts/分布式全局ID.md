---
title: 分布式全局ID
date: 2021-07-18 17:22:17
categories: 分布式系统
tags:
- 分布式全局ID
---

# 1 前言

在分库分表的过程中，因为拆分的实体表的ID有可能是重复的，正式由于这个问题才会有分布式全局ID这个方案的出现。

<!-- more -->

# 2 概要

- 分库分表的系统中，由于id引发的问题

  它是由于什么原因导致的？它会对业务系统引发什么问题？

- 分布式ID的解决方案

  1. 使用UUID作为id实现主键全局ID唯一性保证
  2. 通过统一ID序列表，实现全局ID
  3. 雪花算法作为全局ID
  4. 多种方案的比较

# 3 分库分表的系统中，由于id引发的问题

它是由于什么原因导致的？它会对业务系统引发什么问题？

- 每个表通常都会有唯一标识，通常使用id
- ID通常采用自增的方式
- 在分库分表的情况下，每张表的id都是从0开始自增的
- 两个分片表中存在相同的 id
- 从而导致业务混乱

# 4 分布式ID的解决方案

## 4.1 UUID 

- UUID 通用唯一识别码（Universally Unique Identifier）。

- 使用 UUID，保证每一条的记录id都是不同的。

- 缺点1：只是单纯的一个id，没有实际意义。长度32位，太长。
- mycat不支持UUID的方式。
- sharding-Jdbc支持UUID的方式。

## 4.2 统一ID序列

- ID 的值统一的从一个集中的ID序列生成器中获取

  ![](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/bH5oQj-20210718211022587.png)

- ID序列生成器MyCat支持，Sharding-Jdbc不支持
- MyCat中有两种方式：本地文件方式、数据库方式
- 本地文件方式用于测试、数据库方式用于生产
- 优点：ID集中管理，避免重复
- 缺点：并发量大时，ID生成器压力较大

## 4.3 雪花算法

- SnowFlake是由Twitter提出的分布式ID算法
- 一个64 bit 的 long 型的数字（是二进制数，在程序中会转化为10进制数）
- 引入了时间戳，保持自增

![](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/ZvQWuo.png)

- 基本保持全局唯一，毫秒内并发最大 4096 个ID
- 时间回调，可能引起ID重复
- MyCat 和 Sharding-Jdbc 均支持雪花算法
- Sharding-Jdbc 可设置最大容忍回调时间
