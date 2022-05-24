---
title: "MySQL事务的4种隔离级别"
date: 2022-04-15T15:47:13+08:00
categories: 数据库
tags:
- mysql
---

# MySQL事务的4种隔离级别

# 1 简介

事务的4种隔离级别分别是读未提交（Read Uncommitted）、读已提交（Read Committed）、 可重复读（Repeatable Read）和串行化（Serializable）。

首先，在了解这4种隔离级别前就必须先要了解其前提，也就是事务，本文简单介绍一下关于事务。

之后，我们也要理解这4种隔离级别产生的原因和场景展现以及4种隔离级别是如何解决问题的。

# 2 什么是数据库事务？

事务由一个有限的数据库操作序列组成，这些操作要么全部执行，要么全部不执行，是一个不可分割的工作单位。

例如一个银行转账场景：

A转账B 100元，A的账号扣除100元，B的账号加上100块。假如中间出现任何异常，例如，在A的账号扣100元时，银行瘫痪，B的账号余额没有发生变化。这时候就需要事务来保证将A的钱还回去。

## 2.1 事务的四大特性（ACID）

- 原子性：事务作为一个整体被执行，包含在其中的对数据库的操作要么全部都执行，要么都不执行。
- 一致性：指在事务开始之前和事务结束以后，数据不会被破坏，假如A账户给B账户转10块钱，不管成功与否，A和B的总金额是不变的。
- 隔离性：多个事务并发访问时，事务之间是相互隔离的，一个事务不应该被其他事务干扰，多个并发事务之间要相互隔离。
- 持久性：表示事务完成提交后，该事务对数据库所作的操作更改，将持久地保存在数据库之中。

# 3 并发事务会导致的问题

- 脏读：事务 A 读取了事务 B 更新的数据，然后 B 进行回滚操作，那么A读取的数据就是脏数据
- 不可重复读：事务A多次读取同一数据，事务B在事务A多次读取的过程中，对数据做了更新并提交，导致事务A多次夺取同一数据时，结果不一致。
- 幻读：系统管理员A将数据库中所有学生的成绩从具体分数改为ABCDE等级，但是系统管理员B就在这个时候插入了一条具体分数的记录，当系统管理员A改结束后发现还有一条记录没有改过来，就好像发生了幻觉一样，这就叫幻读。

>💡 不可重复读的和幻读很容易混淆，不可重复读侧重于修改，幻读侧重于新增或删除。解决不可重复读的问题只需锁住满足条件的行，解决幻读需要锁表。


## 3.1 本文会使用到的 SQL 语句：

### 3.1.1 示例表结构

```sql
CREATE TABLE `account` (
  `id` int(11) NOT NULL,
  `name` varchar(255) DEFAULT NULL,
  `balance` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `un_name_idx` (`name`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### 3.1.2 查询事务的默认隔离级别

```sql
mysql> select @@transaction_isolation;
+-------------------------+
| @@transaction_isolation |
+-------------------------+
| REPEATABLE-READ         |
+-------------------------+
1 row in set (0.01 sec)
```

### 3.1.3 设置当前会话的事务隔离级别

```sql
mysql> set session transaction isolation level read uncommitted;
Query OK, 0 rows affected (0.00 sec)
```

# 4 事务的4种隔离级别和示例演示

| 事务隔离级别                             | 脏读 | 不可重复读 | 幻读 |
| ---------------------------------------- | ---- | ---------- | ---- |
| 读未提交（read-uncommitted）             | 是   | 是         | 是   |
| 不可重复读（read-committed）又叫读已提交 | 否   | 是         | 是   |
| 可重复读（repeatable-read）              | 否   | 否         | 是   |
| 串行化（serializable）                   | 否   | 否         | 否   |

## 4.1 读未提交

事务A：

![Untitled](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/mysql_transaction_0.png)

事务B：

![Untitled](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/mysql_transaction_1.png)

>💡 读未提交是隔离级别最低的，会造成脏读。


## 4.2 读已提交

为了避免脏读，数据库有了比**读未提交**更高的隔离级别，即**读已提交**。

对于提交：当前事务只能读取其它事务已提交的数据，未提交事务的数据读取不到。

事务A：

![Untitled](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/mysql_transaction_2.png)

事务B：

![Untitled](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/mysql_transaction_3.png)

由此可以得出结论，隔离级别设置为**已提交读（READ COMMITTED）**
 时，已经不会出现脏读问题了，当前事务只能读取到其他事务提交的数据。但是，站在事务A的角度想想，存在其他问题吗？

### **提交读的隔离级别会有什么问题呢？**

在同一个事务A里，相同的查询sql，读取同一条记录（id=1），读到的结果是不一样的，即**不可重复读**。所以，隔离级别设置为`read committed`的时候，还会存在**不可重复读**的并发问题。

## 4.3 可重复读

为了避免**不可重复读**的并发问题，我们将隔离级别设置为**可重复读（REPEATABLEE READ）**，重复一下之前的操作。

事务A：

![Untitled](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/mysql_transaction_4-20220415155652729.png)

事务B：

![Untitled](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/mysql_transaction_5.png)

https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/mysql_transaction_5.png

到了这一步，可以发现事务隔离级别设置为可重复读，可以解决幻读问题。

### 那么**可重复读**真的是否已经解决了幻读问题呢？毕竟还剩个事务隔离级别呢。

RR隔离级别下，手动启动一个事务，进行select操作，他会生成一个快照，可以理解为将当前数据库的数据复制一份，在当前事务中，之后不管进行多少次select查询，都是在模板中去取数据，所以不管数据库中是否对数据进行了改变，都不会影响当前事务数据的读取，从而避免了幻读。**这种普通的 select 操作，称为快照读**。

但是如果在当前事务中使用了下图语句进行**当前读**:

```sql
select * from account for update;
```

`for update`是进行**当前读**的操作，他会重新从数据库去加载当前的最新的数据，每执行一次加载一次，如果在此时，另外一个事务为数据库添加了一个事务，再进行查询，会发现查询的数据与之前相比多了或者少了，这也就是幻读现象。

如果你阅读到这里，去实操一下，会发现和我说的不一样，有一种上当的感觉。

其实不是的，这是因为上述都是在标准的可重复读下的情况，在innodb存储引擎中对可重复读进行了改造，为当前读加上了 `Next-key Lock`,也就是间隙锁和行锁的统称，**行锁防止了别的事务修改或者删除，间隙锁防止了别的事务新增**。也就是在进行上面的`for update`事务中，其他的事务不能对数据进行增删操作，执行会报错或者长时间处于等待状态。

>💡 注意：如果A事务如果进行了快照读，然后通过B事务对数据就行增删，然后紧接着A事务进行当前读操作，两次读取数据不一致，不能算作幻读，因为幻读定义是同一个select语句，快照读和当前读的查询语句是不一样的.

### 小结

1. **数据库的并发问题有：脏读、不可重复读和幻读；**
2. **事务隔离级别依次为：读未提交、读已提交、可重复读和串行化；**
3. **在标准的RR下并没有彻底解决幻读，但是在Mysql的innodb引擎中彻底解决了；**
4. **innodb通过** Next-Key lock**解决的幻读问题，其实也就是阻塞串行化了；**
5. **不能把快照读和当前读在一个事务中进行比较是否出现幻读，两者不是同一个select，不满足幻读的官方定义。**

## 4.4 串行化

略，这部分我懒得放图了，因为结果和上面没啥差别。

# 相关性

---

# 文献引用

---

[一文彻底读懂MySQL事务的四大隔离级别 - Jay_huaxiao - 博客园](https://www.cnblogs.com/jay-huaxiao/p/12639435.html)

[MySQL的四种事务隔离级别](https://www.cnblogs.com/yaochunhui/p/14177906.html)

[关于数据库隔离级别为RR(可重复读)下是否解决幻读问题_眉梢i的博客-CSDN博客_rr解决幻读](https://blog.csdn.net/qq_45225798/article/details/119978841)

