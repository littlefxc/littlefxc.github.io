---
title: hive 2.3.7 安装
date: 2020-09-22 11:10:05
categories: 大数据
tags: hive,大数据
---

# Hive介绍

官网：[hive.apache.org/](http://hive.apache.org/)

Apache Hive™**数据仓库**软件有助于使用SQL读取，编写和管理驻留在**分布式存储**中的**大型数据集**。可以将结构投影到已存储的数据中。提供了**命令行工具**和**JDBC驱动程序**以将用户连接到Hive。

hive提供了SQL查询功能 hdfs分布式存储。

hive本质HQL转化为MapReduce程序。

# Hive 安装前提

1. 启动 hdfs 集群
2. 启动 yarn 集群
3. 启动 mysql

如果想用hive的话，需要提前安装部署好hadoop集群。

# 安装 Hive

```shell
ln -s /opt/apache-hive-2.3.7-bin /usr/local/hive
```

# 编辑环境变量

```shell
vi /etc/profile
```

```bash
HIVE_HOME=/usr/local/hive
export HIVE_HOME
PATH=$JAVA_HOME/bin:$HIVE_HOME/bin
```

# 配置 hive-site.xml

```shell
cd /usr/local/hive
vi conf/hive-site.xml
```

```xml
<configuration>
  <property>
      <name>javax.jdo.option.ConnectionURL</name>
      <value>jdbc:mysql://master:3306/hive?createDatabaseIfNotExist=true&amp;useSSL=true</value>
      <description>JDBC connect string for a JDBC metastore</description>    
  </property>
  <property> 
      <name>javax.jdo.option.ConnectionDriverName</name> 
      <value>com.mysql.jdbc.Driver</value> 
      <description>Driver class name for a JDBC metastore</description>     
   </property>               

   <property> 
      <name>javax.jdo.option.ConnectionUserName</name>
      <value>root</value>
      <description>username to use against metastore database</description>
   </property>
   <property>  
      <name>javax.jdo.option.ConnectionPassword</name>
      <value>123456</value>
      <description>password to use against metastore database</description>  
   </property>
</configuration>
```

下面的部分如果不配置会产生错误

```xml
<property>
  <name>hive.exec.local.scratchdir</name>
  <value>/usr/local/hive</value>
  <description>Local scratch space for Hive jobs</description>
</property>
 
<property>
  <name>hive.downloaded.resources.dir</name>
  <value>/usr/local/hive/hive-downloaded-addDir/</value>
  <description>Temporary local directory for added resources in the remote file system. </description>
</property>
 
<property>
  <name>hive.querylog.location</name>
  <value>/usr/local/hive/querylog-location-addDir/</value>
  <description>Location of Hive run time structured log file</description>
</property>
 
<property>
  <name>hive.server2.logging.operation.log.location</name>
  <value>/usr/local/hive/hive-logging-operation-log-addDir/</value>
  <description>Top level directory where operation logs are stored if logging functionality is enabled</description>
</property>

<!-- hiveserver2 的配置 -->
<property>
  <name>hive.server2.thrift.port</name>
  <value>10000</value>
</property>
<property>
  <name>hive.server2.thrift.bind.host</name>
  <value>localhost</value>
</property>
```



# 配置 hive-env.sh

```bash
# 添加
export JAVA_HOME=/usr/local/java
export HIVE_HOME=/usr/local/hive
export HADOOP_HOME=/usr/local/hadoop
export HIVE_CONF_DIR=/usr/local/hive/conf
```

# 修改hive-log4j.properties

```shell
vi hive-log4j.properties
```

```bash
hive.log.dir=自定义目录
```

# 下载并配置 mysql 驱动包

```shell
cp /mnt/share/mysql-connector-java-5.1.47.jar /usr/local/hive/lib/
```



# 初始化元数据

```shell
/usr/local/hive/bin/schematool -dbType mysql -initSchema
```

# 启动 Hive

```shell
/usr/local/hive/bin/hive
```

# 测试

```shell
create table users(user_id int,username varchar(20),pwd varchar(20),email varchar(30),grade int);
insert into users(user_id,username,pwd,email,grade)values(1,'admin','1234','admin@qq.com',2);
insert into users(user_id,username,pwd,email,grade)values(2,'admin2','1234','admin2@qq.com',2);

```

