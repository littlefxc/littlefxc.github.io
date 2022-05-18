---
title: 基于Sharding-Jdbc的读写分离和分库分表
date: 2021-07-11 11:37:33
tags:
- 分库分表
- 读写分离
- sharing-jdbc
---

# 简介

- 是一个开源的分布式的关系型数据库的中间件
- 已于2020年4月16日成为 Apache 软件基金会的顶级项目
- 客户端代理模式
- 定位为轻量级的Java框架，以 jar 包提供服务
- 可以理解为增强版的 jdbc 驱动
- 完全兼容各种 ORM 框架

<!-- more -->

## 架构图

![ShardingSphere-JDBC Architecture](https://shardingsphere.apache.org/document/current/img/shardingsphere-jdbc-brief.png)

## 配置方式

- Java API
- Yaml
- Spring Boot
- Spring

## 与 Mycat 的区别

- MyCat 是服务端的代理，Sharding-Jdbc 是客户端代理

- MyCat 不支持同一库内的水平切分，Sharding-Jdbc 支持

