---
title: <SQL反模式>
date: 2019-06-20 14:47:47
categories: SQL
tags: 阅读笔记
---

## 《SQL反模式》 阅读笔记

### 反模式分类

- 逻辑数据库设计反模式
- 物理书库设计反模式
- 查询反模式
- 应用程序开发反模式

### 反模式分解

- 目的
- 反模式
- 如何识别反模式
- 合理使用反模式
- 解决方案

### ER 图示例

![ER 图示例]()

### 范例数据库

`SERIAL` 是 MySQL 中 `BIGINT UNSIGNED NOT NULL AUTO_INCREMENT UNIQUE` 的别名

```sql
CREATE TABLE Accounts (
    account_id         SERIAL PRIMARY KEY,
    account_name       VARCHAR(20),
    first_name         VARCHAR(20),
    last_name          VARCHAR(20),
    email              VARCHAR(100),
    password_hash      CHAR(64),
    portrait_image     BLOB,
    hourly_rate        NUMERIC(9,2)
);

CREATE BugStatus (
    status VARCHAR(20) PRIMARY KEY
);

CREATE TABLE Bugs (
    bug_id             SERIAL PRIMARY KEY,
    date_reported      DATE NOT NULL,
    summary            VARCHAR(80),
    description        VARCHAR(1000),
    resolition         VARCHAR(1000),
    reported_by        BIGINT UNSINGNED NOT NULL,
    assigned_to        BIGINT UNSINGNED,
    verified_by        BIGINT UNSINGNED,
    status             VARCHAR(20) NOT NULL DEFAULT 'NEW',
    priority           VARCHAR(20),
    hours              NUMERIC(9,2),
    FOREIGN KEY (reported_by) REFERENCES Accounts(account_id),
    FOREIGN KEY (assigned_to) REFERENCES Accounts(account_id),
    FOREIGN KEY (verified_by) REFERENCES Accounts(account_id),
    FOREIGN KEY (status) REFERENCES BugStatus(status)
);

CREATE TABLE Comments (
    comment_id         SERIAL PRIMARY KEY,
    bug_id             BIGINT UNSINGNED NOT NULL,
    author             BIGINT UNSINGNED NOT NULL,
    comment_date       DATETIME NOT NULL,
    comment            TEXT NOT NULL,
    FOREIGN KEY        TEXT NOT NULL,
);
```
