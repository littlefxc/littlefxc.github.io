---
title: MySQL主从搭建
date: 2021-06-20 16:45:19
categories: mysql
tags:
- 读写分离
- mysql
---

# 关键点

- 主配置 log-bin，指定文件的名字

- 主配置 server-id，默认为1

- 从 server-id 与主不能重复

- 主数据库创建备份账户并授权 `REPLICATION SLAVE`

- 主数据库锁表 `FLUSH TABLES WITH READ LOCK`

- 主数据库找到 `log-bin` 的位置 `SHOW MASTER STATUS`

- 备份主数据库数据 `mysqldump -all-datables --master-data > dbduump.db`

- 主数据库解锁 `unlock tables`

- 从数据库导入 dump的数据

- 在从数据库上设置主数据库的配置

  ```sql
  mysql> CHANGE MASTER TO
  	-> MASTER_HOST='master_host_name', 	
  	-> MASTER_PORT=port_num 
  	-> MASTER_USER='replication_user_name', 
  	-> MASTER_PASSWORD='replication_password', 			        
  	-> MASTER_LOG_FILE='recorded_log_file_name',			   
    -> MASTER_LOG_POS=recorded_log_position;                                                       
  ```

  - master_host_name : MySQL主的地址
  - port_num : MySQL主的端口（数字型）
  - replication_user_name : 备份账户的用户名
  - replication_password : 备份账户的密码
  - recorded_log_file_name ：bin-log的文件名
  - recorded_log_position : bin-log的位置（数字型）
  - bin-log的文件名和位置 是 从 `show master status` 得到的。

<!-- more -->

# 配置文件模版

## 主 mysql 的配置:master.cnf

```cnf
[mysqld]
## 设置server_id，一般设置为IP，注意要唯一
server_id=100
## 复制过滤：也就是指定哪个数据库不用同步（mysql库一般不同步）
binlog-ignore-db=mysql
## 开启二进制日志功能，可以随便取，最好有含义（关键就是这里了）
log-bin=replicas-mysql-bin
## 为每个session分配的内存，在事务过程中用来存储二进制日志的缓存
binlog_cache_size=1M
## 主从复制的格式（mixed,statement,row，默认格式是statement）
binlog_format=mixed
## 二进制日志自动删除/过期的天数。默认值为0，表示不自动删除。
expire_logs_days=7
## 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断。
## 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致
slave_skip_errors=1062
```

## 从 mysql 的配置:slave.cnf

```cnf
[mysqld]
## 设置server_id，一般设置为IP，注意要唯一
server_id=101
## 复制过滤：也就是指定哪个数据库不用同步（mysql库一般不同步）
binlog-ignore-db=mysql
## 开启二进制日志功能，以备Slave作为其它Slave的Master时使用
log-bin=replicas-mysql-slave1-bin
## 为每个session 分配的内存，在事务过程中用来存储二进制日志的缓存
binlog_cache_size=1M
## 主从复制的格式（mixed,statement,row，默认格式是statement）
binlog_format=mixed
## 二进制日志自动删除/过期的天数。默认值为0，表示不自动删除。
expire_logs_days=7
## 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断。
## 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致
slave_skip_errors=1062
## relay_log配置中继日志
relay_log=replicas-mysql-relay-bin
## log_slave_updates表示slave将复制事件写进自己的二进制日志
log_slave_updates=1
## 防止改变数据(除了特殊的线程)
read_only=1
```

# 主数据库创建备份账户并授权 `REPLICATION SLAVE`

```sql
create user 'repl'@'%' identified by 'Fengxuechao@13579'
grant replication slave on *.* to 'repl'@'%';
flush privileges;
```

