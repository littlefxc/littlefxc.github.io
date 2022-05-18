---
title: 搭建Mycat
date: 2021-06-09 20:51:05
categories: 数据库
tags:
- mycat
---

# 1 环境搭建

1. 3台服务器
2. centos 7
3. 采用 yum 方式，在其中两台安装 mysql
4. 检查mysql 安装是否正确
5. 下载 Mycat 软件包
6. 在第3台机器上安装mycat，并修改配置文件

   ![image-20210609210635587](https://gitee.com/littlefxc/oss/raw/master/images/image-20210609210635587.png)

7. 连接mycat，体验数据的增删改查

# 2 mysql 安装教程

## 2.1 查询是否安装了mysql**

  ```
  rpm -qa|grep mysql                                                      
  ```

## 2.2 卸载mysql （下面是卸载mysql的库，防止产生冲突，mysql也是类似卸载方式）**

```
rpm -e --nodeps mysql-libs-5.1.*
卸载之后，记得：
find / -name mysql
删除查询出来的所有东西
```

## 2.3 安装mysql

```
yum install mysql-server                                                    
```

注意: centos 7这样安装不行, 详见文档底部

## 2.4 启动mysql

```
启动方式1：service mysql start
启动方式2：/etc/init.d/mysql start
启动方式3：service mysqld start
启动方式4：/etc/init.d/mysqld start                                               
```

## 2.5 root账户默认是没有密码的，修改root密码：

```
/usr/bin/mysqladmin -u root password 密码 
例如：
/usr/bin/mysqladmin -u root password pwd    这样就将root密码设置成pwd了       
```

## 2.6 重置root密码（忘记root密码找回）

### 2.6.1 停止MySQL服务命令:

```
/etc/init.d/mysqld stop 
/etc/init.d/mysql stop                                                        
```

### 2.6.2 输入绕过密码认证命令：

```
mysqld_safe --user=mysql --skip-grant-tables --skip-networking &
```

### 1.6.3 输入登录用户命令：

```
mysql -u root mysql                                                        
```

### 2.6.4 输入修改root密码SQL语句：

```
update user set Password=password ('123456') where user='root';          
```

### 2.6.5 输入数据刷新命令：

```
FLUSH PRIVILEGES;                                                   
```

### 2.6.6 退出MySQL命令：

```
quit;                                                              
```

## 2.7 设置允许远程连接

```
grant all privileges on *.* to root@'%' identified by '123456789' with grant option;  
```

## 2.8 开放端口3306，否则依然无法过远程

### 2.8.1 打开防火墙配置文件：

```
vi /etc/sysconfig/iptables                                                   
```

### 2.8.2 添加下面一行：

```
-A INPUT -m state --state NEW -m tcp -p tcp --dport 3306 -j ACCEPT
```

注意：开通3306 端口的行必须在icmp-host-prohibited前，否则无效：以下为配置结果图：

![图片描述](https://gitee.com/littlefxc/oss/raw/master/images/5e81a3ba087c42ce05540189.jpg)

### 2.8.3 重启防火墙，使配置生效：

```
/etc/init.d/iptables restart                                                
```

## 2.9 设置开机启动mysql：

### 2.9.1 查看MySQL服务是否自动开启命令

```
 chkconfig --list | grep mysqld
 chkconfig --list | grep mysql                                            
```

### 2.9.2 开启MySQL服务自动开启命令

```
chkconfig mysqld on
chkconfig mysql on                                              
```

## 2.10 将mysql默认引擎设置为InnoDB

修改MySQL配置文件my.cnf

```
cd /etc
vi my.cnf                                                         
```

在[mysqld]一段加入

```
default-storage-engine=InnoDB                                            
```

删除ib_logfile0、ib_logfile1两个文件

```
cd /var/lib/mysql
rm -rf ib_logfile*                                                   
```

重启mysql

## 2.11 开启mysql的日志(监控执行的sql语句)

命令: `show global variables like ‘%general%’;` 该语句可以查看是否开启, 以及生成的位置

```
set global general_log = on; // 打开  
set global general_log = off; // 关闭      
```

参考文档:

[http://blog.csdn.net/fdipzone/article/details/16995303](http://blog.csdn.net/fdipzone/article/details/16995303)

## 2.12 centos7安装mysql

```
wget http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm
#rpm -ivh mysql-community-release-el7-5.noarch.rpm
#yum install mysql-community-server
成功安装之后重启mysql服务
#service mysqld restart
初次安装mysql是root账户是没有密码的
设置密码的方法
#mysql -uroot
mysql> set password for ‘root’@‘localhost’ = password('mypasswd');
mysql> exit
```

# 3 Mycat 安装教程

[mycat官网](http://www.mycat.org.cn)

[mycat1权威指南](https://www.yuque.com/ccazhw/tuacvk)

[mycat2权威指南](https://www.yuque.com/ccazhw/ml3nkf)

## 3.1 安装 Mycat

```sh
tar -zxvf Mycat-server-1.6.7.1-release-20200209222254-mac.tar
```

## 3.2 启动和验证

```sh
bin/mycat start
```

启动后，验证一些基本操作，如下图所示：

![image-20210610162658349](https://gitee.com/littlefxc/oss/raw/master/images/image-20210610162658349.png)

可以看到，我们成功连上了 mycat 服务器，MyCat 服务器默认定义了一个名为 TESTDB 的逻辑数据库，并且也在该逻辑数据库中定义了一些逻辑表。

但当我们尝试做一些 select 操作的时候，控制台会提示报错，这是因为 MyCat 配置错误导致的。

所以我们需要进行配置。

## 3.3 配置

![image-20210610163531656](https://gitee.com/littlefxc/oss/raw/master/images/image-20210610163531656.png)

其中 ,

- bin 目录是 MyCat 的启动目录，
- conf 目录是 MyCat 的配置文件目录，
- lib 目录是 MyCat 自身的 Jar 包以及所依赖 Jar 包的目录，
- logs 目录是日志目录。

在 conf 目录下有 3 个重要的配置文件：

- schema.xml
- Server.xml
- Rule.xml

下面就来简单说明这 3 个配置文件的关键配置

### 3.3.1 schema.xml

schema.xml 文件定义了 MyCat 到底连接那个数据库实例，连接这个数据库实例的哪个数据库。MyCat 一共有几个逻辑数据库，MyCat 一共有几个逻辑表。

schema.xml 文件一共有四个配置节点：`DataHost`、`DataNode`、`Schema`、`Table`。

- DataHost：定义数据库实例

  - balance：负载均衡类型
    - balance="0", 不开启读写分离机制，所有读操作都发送到当前可用的writeHost上。
    - balance="1"，全部的readHost与stand by writeHost参与select语句的负载均衡，简单的说，当双主双从模式(M1->S1，M2->S2，并且M1与 M2互为主备)，正常情况下，M2,S1,S2都参与select语句的负载均衡。
    - balance="2"，所有读操作都随机的在writeHost、readhost上分发。
    - balance="3"，所有读请求随机的分发到wiriterHost对应的readhost执行，writerHost不负担读压力，注意balance=3只在1.4及其以后版本有，1.3没有。

  - writeType：写请求类型，0落在第一个writeHost上；1随机；

- DataNode：定义数据库名称

- Schema：定义逻辑库

  - checkSQLschema：是否去掉SQL中的schema

  - sqlMaxLimit：select 默认的`limit`值，仅对分片表有效
  - rule：定义分片表的分片规则，必须与`rule.xml`中的`tableRule`对应
  - ruleRequired：是否绑定分片规则，如果为true，没有绑定分片规则，程序报错

- Table：定义逻辑表

DataHost 节点定义了 MyCat 要连接哪个 MySQL 实例，连接的账号密码是多少。默认的 MyCat 为我们定义了一个名为 localhost1 的数据服务器（DataHost），它指向了本地（localhost）3306 端口的 MySQL 服务器，对应 MySQL 服务器的账号是 root，密码是 123456。

  ```xml
  <dataHost name="localhost1" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
  	<heartbeat>select user()</heartbeat> 
  	<writeHost host="hostM1" url="localhost:3306" user="root" password="123456"> 
  		<readHost host="hostS2" url="192.168.1.200:3306" user="root" password="xxx" />
  	</writeHost>
  	<writeHost host="hostS1" url="localhost:3316" user="root" password="123456" /> 
  </dataHost>
  ```

DataNode 节点指定了需要连接的具体数据库名称，其使用一个 dataHost 属性指定该数据库位于哪个数据库实例上。默认的 MyCat 为我们创建了三个数据节点（DataNode），dn1 数据节点对应 localhost1 数据服务器上的 db1 数据库，dn2 数据节点对应 localhost1 数据服务器上的 db2 数据库，dn1 数据节点对应 localhost1 数据服务器上的 db3 数据库。

  ```xml
  <dataNode name="dn1" dataHost="localhost1" database="db1" />
  <dataNode name="dn2" dataHost="localhost1" database="db2" />
  <dataNode name="dn3" dataHost="localhost1" database="db3" />
  ```

Schema 节点定义了 MyCat 的所有逻辑数据库，Table 节点定义了 MyCat 的所有逻辑表。默认的 MyCat 为我们定义了一个名为 TESTDB 的逻辑数据库，在这个逻辑数据库下又定义了名为 travaelrecord、company 等 6 个逻辑表。

  ```xml
  <schema name="TESTDB" checkSQLschema="false" sqlMaxLimit="100">
  		<!-- auto sharding by id (long) -->
  		<table name="travelrecord" dataNode="dn1,dn2,dn3" rule="auto-sharding-long" />
  
  		<!-- global table is auto cloned to all defined data nodes ,so can join
  			with any table whose sharding node is in the same data node -->
  		<table name="company" primaryKey="ID" type="global" dataNode="dn1,dn2,dn3" />
  		<table name="goods" primaryKey="ID" type="global" dataNode="dn1,dn2" />
  		<!-- random sharding using mod sharind rule -->
  		<table name="hotnews" primaryKey="ID" autoIncrement="true" dataNode="dn1,dn2,dn3"
  			   rule="mod-long" />
  		<!-- <table name="dual" primaryKey="ID" dataNode="dnx,dnoracle2" type="global"
  			needAddLimit="false"/> <table name="worker" primaryKey="ID" dataNode="jdbc_dn1,jdbc_dn2,jdbc_dn3"
  			rule="mod-long" /> -->
  		<table name="employee" primaryKey="ID" dataNode="dn1,dn2"
  			   rule="sharding-by-intfile" />
  		<table name="customer" primaryKey="ID" dataNode="dn1,dn2"
  			   rule="sharding-by-intfile">
  			<childTable name="orders" primaryKey="ID" joinKey="customer_id"
  						parentKey="id">
  				<childTable name="order_items" joinKey="order_id"
  							parentKey="id" />
  			</childTable>
  			<childTable name="customer_addr" primaryKey="ID" joinKey="customer_id"
  						parentKey="id" />
  		</table>
  		<!-- <table name="oc_call" primaryKey="ID" dataNode="dn1$0-743" rule="latest-month-calldate"
  			/> -->
  	</schema>
  ```

所以上面当我们登陆 MyCat 输入`show databases`会看到只有一个名为 TESTDB 的数据库，这个就是 MyCat 的逻辑数据库。

### 3.3.2 server.xml

server.xml 定义了项目中连接 MyCat 服务器所需要的账号密码，以及该账号能访问那些逻辑数据库。 server.xml 配置文件中有 `System` 和 `User` 两个配置节点。

System 节点定义了连接 MyCat 服务器的系统配置信息。例如是否开启实时统计功能，是否开启全加班一致性检测等。

```xml
<system>
	<property name="useSqlStat">0</property>  <!-- 1为开启实时统计、0为关闭 -->
	<property name="useGlobleTableCheck">0</property>  <!-- 1为开启全加班一致性检测、0为关闭 --> 
	<property name="sequnceHandlerType">2</property> 
	<property name="processorBufferPoolType">0</property> 
	……
</system>
```

User 配置节点定义了连接 MyCat 服务器的账号密码，以及该账号密码所能进行的数据库操作。默认的 MyCat 为我们创建了一个账户名为 root，密码为 123456 的账号，只能访问 TESTDB 逻辑数据库，并且定义了对相关表的操作权限。

```xml
<user name="root">
	<property name="password">123456</property>
	<property name="schemas">TESTDB</property> 
	<privileges check="false">
		<schema name="TESTDB" dml="0110" >
			<table name="tb01" dml="0000"></table>
			<table name="tb02" dml="1111"></table>
		</schema>
	</privileges>	 
</user>
```

### 3.3.3 rule.xml

rule.xml 定义了逻辑表使用哪个字段进行拆分，使用什么拆分算法进行拆分。rule.xml 中有两个配置节点，分别是：`TableRule` 和 `Function` 配置节点。

TableRule 配置节点定义了逻辑表的拆分信息，例如使用哪个字段进行拆分，使用什么拆分算法。默认的 MyCat 为我们配置了一个名为 rule2 的表拆分规则，表示根据 user_id 字段进行拆分，拆分算法是 func1。

```xml
<tableRule name="rule2">
	<rule>
		<columns>user_id</columns>
		<algorithm>func1</algorithm>
	</rule>
</tableRule>
```

Function 配置节点则定义了具体的拆分算法。例如使用对 1000 取余的拆分算法，对 100 取余的拆分算分等等。默认的 MyCat 为我们定义了一个名为` func1` 的**拆分算法**，这个拆分算法定义在 `io.mycat.route.function.PartitionByLong` 类中，并且还传入了两个参数值。

```xml
<function name="func1" class="io.mycat.route.function.PartitionByLong">
	<property name="partitionCount">8</property>
	<property name="partitionLength">128</property>
</function>
```

## 3.4 Mycat FAQ

### 3.4.1 ERROR 1184 (HY000): Invalid DataSource:1

具体错误如下：

![image-20210611093234112](https://gitee.com/littlefxc/oss/raw/master/images/image-20210611093234112.png)

错误原因有两种可能：

1. 没有为mysql用户配置远程访问的权限

   ```xml
   <writeHost host="db1" url="192.168.0.3:3306" user="root" password="123456" />
   ```

   授予mysql用户远程访问的权限

   ```sql
   GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456'

2. 没有在mysql中创建数据库

   ```sql
   # 此操作在当前机的mysql上操作（不再mycat）
   # mysql -uroot -p
   CREATE DATABASE IF NOT EXISTS mycatdb1 DEFAULT CHARSET utf8mb4 COLLATE utf8mb4_general_ci;
   CREATE DATABASE IF NOT EXISTS mycatdb2 DEFAULT CHARSET utf8mb4 COLLATE utf8mb4_general_ci;
   CREATE DATABASE IF NOT EXISTS mycatdb3 DEFAULT CHARSET utf8mb4 COLLATE utf8mb4_general_ci;
   
   # 三个分库各自创建表travelrecord
   CREATE TABLE `travelrecord` (
     `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
     `name` varchar(22) NOT NULL DEFAULT '',
     `time` int(10) unsigned NOT NULL DEFAULT '0',
     PRIMARY KEY (`id`)
   ) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4;
   
   # 模拟数据
   INSERT INTO `mycat-db1`.`travelrecord` (`name`, `time`) VALUES ('qkl', '0');
   INSERT INTO `mycat-db1`.`travelrecord` (`name`, `time`) VALUES ('andy', '0');
   INSERT INTO `mycat-db2`.`travelrecord` (`name`, `time`) VALUES ('zgq', '0');
   INSERT INTO `mycat-db3`.`travelrecord` (`name`, `time`) VALUES ('pcb', '0');
   ```

   在 mycat 的配置文件 schema.xml 中修改 `dataNode`节点：

   ```xml
   <dataNode name="dn1" dataHost="localhost1" database="mycatdb1" />
   <dataNode name="dn2" dataHost="localhost1" database="mycatdb2" />
   <dataNode name="dn3" dataHost="localhost1" database="mycatdb3" />
   ```

   

