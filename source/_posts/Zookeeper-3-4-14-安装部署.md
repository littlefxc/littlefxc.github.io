---
title: Zookeeper 3.4.14 安装部署
date: 2020-09-15 20:09:07
categories: 大数据
tags: zookeeper,大数据
---

[TOC]

#### 下载

> 下载地址：http://zookeeper.apache.org/

下载过程就不说了，我们下载了最新的`zookeeper-3.4.14`。

#### 安装

**1、上传安装包**

把下载的最新的包（如：zookeeper-3.4.14.tar.gz）上传到服务器，上传的方式也不多说了。

**2、解压**

```sh
$ tar zxvf zookeeper-3.4.14.tar.gz
```

**3、移动到/usr/local目录下**

```sh
$ ln -s zookeeper-3.4.14 /usr/local/zookeeper
```

### 服务器分配

| 主机名       | IP             |
| ------------ | -------------- |
| centos-node1 | 192.168.99.101 |
| centos-node2 | 192.168.99.102 |
| centos-node3 | 192.168.99.103 |

#### 集群配置

Zookeeper集群原则上需要2n+1个实例才能保证集群有效性，所以集群规模至少是3台。

下面演示如何创建3台的Zookeeper集群，N台也是如此。

**1、创建数据文件存储目录**

```sh
$ cd /usr/local/zookeeper
$ mkdir data
```

**2、添加主配置文件**

```sh
$ cd conf
$ cp zoo_sample.cfg zoo.cfg
```

**3、修改配置**

```sh
$ vi zoo.cfg
```

先把`dataDir=/tmp/zookeeper`注释掉，然后添加以下核心配置。

```sh
dataDir=/usr/local/zookeeper/data
server.1=centos-node1:2888:3888
server.2=centos-node2:2888:3888
server.3=centos-node3:2888:3888
```

**4、创建myid文件**

```sh
$ cd ../data
$ touch myid
$ echo "1">>myid
```

每台机器的myid里面的值对应server.后面的数字x。

**5、开放3个端口**

使用 iptables

```sh
$ sudo /sbin/iptables -I INPUT -p tcp --dport 2181 -j ACCEPT
$ sudo /sbin/iptables -I INPUT -p tcp --dport 2888 -j ACCEPT
$ sudo /sbin/iptables -I INPUT -p tcp --dport 3888 -j ACCEPT

$ sudo /etc/rc.d/init.d/iptables save
$ sudo /etc/init.d/iptables restart

$ sudo /sbin/iptables -L -n
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           tcp dpt:3888 
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           tcp dpt:2888 
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           tcp dpt:2181
```

或者 firewalld

```sh
# 查看当前区域
$ firewall-cmd --get-active-zones
# 新建一个自定义服务
$ firewall-cmd --new-service=zookeeper --permanent
$ firewall-cmd --service=zookeeper --add-port 2181/tcp --permanent
$ firewall-cmd --service=zookeeper --add-port 2888/tcp --permanent
$ firewall-cmd --service=zookeeper --add-port 3888/tcp --permanent
# 不中断服务的重新加载
$ firewall-cmd --reload
$ firewall-cmd --add-service=zookeeper
# 将当前防火墙的规则永久保存；
$ firewall-cmd --runtime-to-permanent
```



**6、配置集群其他机器**

把配置好的Zookeeper目录复制到其他两台机器上，重复上面4-5步。

```sh
$ scp -r /usr/local/zookeeper centos-node2:/usr/local/
```

```sh
$ vi /etc/profile 
```

```bash
JAVA_HOME=/usr/local/java/
JRE_HOME=$JAVA_HOME/jre
HADOOP_HOME=/usr/local/hadoop
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
ZOOKEEPER_HOME=/usr/local/zookeeper

PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin:$HADOOP_HOME/bin:$ZOOKEEPER_HOME/bin

export JAVA_HOME JRE_HOME PATH CLASSPATH HADOOP_HOME HBASE_HOME
```

**7、重启集群**

```sh
$ /usr/local/zookeeper/bin/zkServer.sh start
```

3个Zookeeper都要启动。

**8、查看集群状态**

```sh
$ /usr/local/zookeeper/bin/zkServer.sh status 
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper/bin/../conf/zoo.cfg
Mode: follower
```

#### 客户端连接

```sh
./zkCli.sh -server 192.168.99.101:2181
```

连接本机的不用带-server。

#### 注意

如果是在单机创建的多个Zookeeper伪集群，需要对应修改配置中的端口、日志文件、数据文件位置等配置信息。