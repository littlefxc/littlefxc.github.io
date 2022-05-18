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
CREATE TABLE IF NOT EXISTS Accounts (
    account_id         SERIAL PRIMARY KEY,
    account_name       VARCHAR(20),
    first_name         VARCHAR(20),
    last_name          VARCHAR(20),
    email              VARCHAR(100),
    password_hash      CHAR(64),
    portrait_image     BLOB,
    hourly_rate        NUMERIC(9,2)
);

CREATE TABLE IF NOT EXISTS BugStatus (
    `status` VARCHAR(20) PRIMARY KEY
);

CREATE TABLE IF NOT EXISTS Bugs (
    bug_id             SERIAL PRIMARY KEY,
    date_reported      DATE NOT NULL,
    summary            VARCHAR(80),
    description        VARCHAR(1000),
    resolition         VARCHAR(1000),
    reported_by        BIGINT UNSIGNED NOT NULL,
    assigned_to        BIGINT UNSIGNED,
    verified_by        BIGINT UNSIGNED,
    `status`           VARCHAR(20) NOT NULL DEFAULT 'NEW',
    priority           VARCHAR(20),
    hours              NUMERIC(9,2),
    FOREIGN KEY (reported_by) REFERENCES Accounts(account_id),
    FOREIGN KEY (assigned_to) REFERENCES Accounts(account_id),
    FOREIGN KEY (verified_by) REFERENCES Accounts(account_id),
    FOREIGN KEY (`status`) REFERENCES BugStatus(`status`)
);

CREATE TABLE IF NOT EXISTS Comments (
    comment_id         SERIAL PRIMARY KEY,
    bug_id             BIGINT UNSIGNED NOT NULL,
    author             BIGINT UNSIGNED NOT NULL,
    comment_date       DATETIME NOT NULL,
    comment            TEXT NOT NULL,
    FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id),
    FOREIGN KEY (author) REFERENCES Accounts(account_id)
);

CREATE TABLE IF NOT EXISTS Screenhots (
    bug_id             BIGINT UNSIGNED NOT NULL,
    image_id           BIGINT UNSIGNED NOT NULL,
    screenshot_image   BLOB,
    caption            VARCHAR(100),
    PRIMARY KEY        (bug_id,image_id),
    FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id)
);

CREATE TABLE IF NOT EXISTS Tags (
    bug_id             BIGINT UNSIGNED NOT NULL,
    tag                VARCHAR(20) NOT NULL,
    PRIMARY KEY        (bug_id,tag),
    FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id)
);

CREATE TABLE IF NOT EXISTS Products (
    product_id         SERIAL PRIMARY KEY,
    product_name       VARCHAR(50) NOT NULL
);

CREATE TABLE IF NOT EXISTS BugsProducts (
    bug_id             BIGINT UNSIGNED NOT NULL,
    product_id         BIGINT UNSIGNED NOT NULL,
    PRIMARY KEY        (bug_id,product_id),
    FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id),
    FOREIGN KEY (product_id) REFERENCES Products(product_id)
);
```
