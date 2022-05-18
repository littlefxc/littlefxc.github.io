---
title: Hadoop 完全分布式安装与部署
date: 2020-09-03 16:10:22
categories: 大数据
tags: hadoop,大数据
---

[Toc]

# Hadoop 2.7.4 完全分布式安装与部署

Hadoop官方指导传送门 [传送门](http://hadoop.apache.org/docs/r2.7.4/hadoop-project-dist/hadoop-common/ClusterSetup.html)

# 服务器准备

服务器规划，提供 3 台服务器，OS 为`centos 7`

| 主机名        | IP             | 预备分配服务                  |
| ------------- | -------------- | --------------------------------- |
| centos-node1 | 192.168.99.101 | DataNode,NodeManager,NameNode |
| centos-node2 | 192.168.99.102 | DataNode,NodeManager,SecondaryNameNode |
| centos-node3 | 192.168.99.103 | DataNode,NodeManager,ResourceManager,HistoryServer |

# 修改主机名

```shell
$ hostnamectl set-hostname centos-node1
```

# 修改服务器静态IP

可以使用 `netstat -r` 来查询网关如下图所示：

![image-20200911142643578](https://gitee.com/littlefxc/oss/raw/master/images/image-20200911142643578.png)

然后将 dhcp 改为 静态IP

```shell
$ vim /etc/sysconfig/network-scripts/ifcfg-enp0s8
```

完全配置如下所示：

```
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
#BOOTPROTO="dhcp"
BOOTPROTO="static"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="enp0s8"
UUID="5b4ea2f4-a5af-4fac-8793-81692730dad9"
DEVICE="enp0s8"
ONBOOT="yes"

# 新增
GATEWAY=192.168.99.0  # 修改网关，虚拟机需要注意修改nat
IPADDR=192.168.99.101 # 分配IP地址
NETMASK=255.255.255.0 # 子网掩码
DNS1=223.5.5.5        # 使用阿里公共DNS1
DNS2=223.6.6.6        # 使用阿里公共DNS2
```



# 修改 hosts

```shell
$ vi /etc/hosts
# 添加
192.168.99.101 centos-node1
192.168.99.102 centos-node2
192.168.99.103 centos-node3
```



# 安装 JDK8

```shell
$ vi /etc/profile
# 添加
JAVA_HOME=/usr/local/java/
JRE_HOME=$JAVA_HOME/jre
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
export JAVA_HOME JRE_HOME PATH CLASSPATH
```



# 增加 dhfs 用户

通常，建议HDFS和YARN以单独的用户身份运行。

在大多数安装中，HDFS进程以 "hdfs" 执行。YARN通常使用 "yarn" 帐户

```sh
$ adduser hdfs
$ passwd hdfs # 修改密码
```

为 `/etc/sudoers`添加如下图所示：

![image-20200911170014063](https://gitee.com/littlefxc/oss/raw/master/images/image-20200911170014063.png)

# 设置 SSH 无密码登录

1. 3 台服务器全部设置

    ```sh
    $ ssh-keygen -t rsa
    ```

2. 各自分配 ssh key

   ```shell
   $ ssh-copy-id centos-node1
   $ ssh-copy-id centos-node2
   $ ssh-copy-id centos-node3
   ```

# 安装部署 Hadoop

## 切换至 hdfs 用户

```shell
$ su - hdfs
```

### 下载

```shell
$ curl -O https://archive.apache.org/dist/hadoop/common/hadoop-2.7.4/hadoop-2.7.4.tar.gz
```

### 解压

```shell
$ tar -zxf hadoop-2.7.4.tar.gz  -C /opt/hadoop-2.7.4
$ ln -s /opt/hadoop-2.7.4 /usr/local/hadoop
$ chown -R hdfs /opt/hadoop-2.7.4
```

### 修改环境变量

```shell
$ sudo vi /etc/profile
# 修改为
JAVA_HOME=/usr/local/java/
JRE_HOME=$JAVA_HOME/jre
HADOOP_HOME=/usr/local/hadoop
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib

PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin:$HADOOP_HOME/bin

export JAVA_HOME JRE_HOME PATH CLASSPATH HADOOP_HOME
```

## 修改 Hadoop 配置

这里我们进入`$HADOOP_HOME`文件夹开始操作

```sh
$ mkdir -p $HADOOP_HOME/hdfs/data
$ mkdir -p $HADOOP_HOME/tmp
```

### 配置hadoop-env.sh

```sh
$ sudo vi $HADOOP_HOME/etc/hadoop/hadoop-env.sh
```

增加 或 修改

```bash
export JAVA_HOME=/usr/local/java/
```

### 配置core-site.xml

```sh
$ sudo vi $HADOOP_HOME/etc/hadoop/core-site.xml
```

configuration配置如下

```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://centos-node1:9000</value>
        <description>HDFS的URI，文件系统://namenode标识:端口号</description>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/usr/local/hadoop/tmp</value>
        <description>namenode上本地的hadoop临时文件夹</description>
    </property>
    <property>     
	    <name>hadoop.proxyuser.root.hosts</name>     
	    <value>*</value>
    </property> 
    <property>     
	    <name>hadoop.proxyuser.root.groups</name>    
      <value>*</value> 
    </property>
    <property>     
    	<name>hadoop.proxyuser.zhaoshb.hosts</name>     
    	<value>*</value> 
    </property> 
    <property>     
    	<name>hadoop.proxyuser.zhaoshb.groups</name>     
    	<value>*</value> 
    </property>
</configuration>
```

配置说明：

`fs.defaultFS`为`NameNode`的地址。 

`hadoop.tmp.dir`为`hadoop`临时目录的地址。默认情况下，`NameNode`和`DataNode`的数据文件都会存在这个目录下的对应子目录下。

### 配置hdfs-site.xml

```sh
$ sudo vi $HADOOP_HOME/etc/hadoop/hdfs-site.xml
```

内容如下：

```xml
<configuration>
    <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>centos-node2:50090</value>
    </property>
    <property>
        <name>dfs.http.address</name>
        <value>centos-node1:50070</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:/usr/local/hadoop/hdfs/name</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:/usr/local/hadoop/hdfs/data</value>
    </property>
    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>
</configuration>
```

配置说明：

`dfs.namenode.secondary.http-address`是指定`secondaryNameNode`的http访问地址和端口号，因为在规划中，我们将`centos-node2`规划为`SecondaryNameNode`服务器。

`dfs.http.address`配置的是本机默认的`dfs`地址，有些服务器可以不用配置，我的试过了，必须加上，不然后续网页打不开。 

`dfs.namenode.name.dir` 指定name文件夹。

`dfs.datanode.data.dir` 指定data文件夹。

 `dfs.datanode.data.dir` 指定副本数，一般小于服务器数，我们设置为`3`

### 配置 slaves

在`hadoop2.x`中叫做`slaves`，在`3.x`版本中改名`workers`。 用来指定`HDFS`上有哪些`DataNode`节点，以及各个节点使用`ip地址`或者`主机名`，用换行分隔。

```sh
$ sudo vi $HADOOP_HOME/etc/hadoop/slaves
```

这里我们就使用主机名

```
centos-node1
centos-node2
centos-node3
```

### 配置yarn-site.xml

```sh
$ sudo vi $HADOOP_HOME/etc/hadoop/yarn-site.xml
```

配置如下

```xml
<configuration>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>centos-node3</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.vmem-check-enabled</name>
        <value>false</value>
    </property>
    <property>
        <name>yarn.log-aggregation-enable</name>
        <value>true</value>
    </property>
    <property>
        <name>yarn.log-aggregation.retain-seconds</name>
        <value>106800</value>
  	</property>
        <name>yarn.application.classpath</name>
    		<value>
            /opt/hadoop-2.7.4/etc/hadoop,
            /opt/hadoop-2.7.4/share/hadoop/common/*,
            /opt/hadoop-2.7.4/share/hadoop/common/lib/*,
            /opt/hadoop-2.7.4/share/hadoop/hdfs/*,
            /opt/hadoop-2.7.4/share/hadoop/hdfs/lib/*,
            /opt/hadoop-2.7.4/share/hadoop/mapreduce/*,
            /opt/hadoop-2.7.4/share/hadoop/mapreduce/lib/*,
            /opt/hadoop-2.7.4/share/hadoop/yarn/*,
            /opt/hadoop-2.7.4/share/hadoop/yarn/lib/*
   		</value>
</configuration>
```

配置说明：

按照规划使用`centos-node3`做为 `resourcemanager` 使用`yarn.log-aggregation-enable`开启日志聚合，`yarn.log-aggregation.retain-seconds`配置聚集的日志在HDFS上最多保存多长时间。

### 配置mapred-site.xml

```sh
$ sudo vi $HADOOP_HOME/etc/hadoop/mapred-site.xml
```

配置如下：

```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>yarn.app.mapreduce.am.env</name>
        <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    <property>
        <name>mapreduce.map.env</name>
        <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    <property>
        <name>mapreduce.reduce.env</name>
        <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    
    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>centos-node3:10020</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.webapp.address</name>
        <value>centos-node3:19888</value>
    </property>
</configuration>
```

配置说明：

`mapreduce.framework.name`设置`mapreduce`任务运行在yarn上。

 `mapreduce.jobhistory.address`是设置`mapreduce`的历史服务器安装在`centos-node3`上。 

`mapreduce.jobhistory.webapp.address`是设置历史服务器的web页面地址和端口号。 

`yarn.app.mapreduce.am.env`,`mapreduce.map.env`,`mapreduce.reduce.env`需要设置为`HADOOP_MAPRED_HOME=${HADOOP_HOME}`，否则在运行yarn程序的时候会出现jar包未找到的错误。

### 修改防火墙

```sh
# 查看当前区域
$ firewall-cmd --get-active-zones
# 新建一个自定义服务
$ firewall-cmd --new-service=hadoop --permanent
$ firewall-cmd --service=hadoop --add-port 4000/tcp --permanent
$ firewall-cmd --service=hadoop --add-port 8088/tcp --permanent
$ firewall-cmd --service=hadoop --add-port 50090/tcp --permanent
$ firewall-cmd --service=hadoop --add-port 50070/tcp --permanent
$ firewall-cmd --service=hadoop --add-port 10020/tcp --permanent
$ firewall-cmd --service=hadoop --add-port 19888/tcp --permanent
# 不中断服务的重新加载
$ firewall-cmd --reload
$ firewall-cmd --add-service=hadoop
# 将当前防火墙的规则永久保存；
$ firewall-cmd --runtime-to-permanent
```



## 启动 Hadoop 集群

完成上述所有必要的配置后，将文件分发到所有服务器的`HADOOP_CONF_DIR`目录下`/usr/local/hadoop/etc/hadoop`。在所有计算机上，该目录应该是相同的目录。

**注意**：启动和停止单个hdfs相关的进程使用的是"hadoop-daemon.sh"脚本，而启动和停止yarn使用的是"yarn-daemon.sh"脚本。

### 格式化

要启动Hadoop集群，需要同时启动`HDFS`和`YARN`集群。 首次启动`HDFS`时，**必须**对其进行格式化。将新的分布式文件系统格式化为`hdfs`.

```sh
$ $HADOOP_HOME/bin/hdfs namenode -format <群集名称>
```

集群名称可以不填写，不出意外，执行完成后`$HADOOP_HOME/hdfs`中就有东西了。

### 启动 HDFS

如果配置了`slaves`和`ssh互信`我们可以

```sh
$ $HADOOP_HOME/sbin/start-dfs.sh
```

### 启动 YARN

如果配置了workers和ssh互信我们可以

```sh
$ $HADOOP_HOME/sbin/start-yarn.sh
```

### 启动 **ResourceManager**

规划在`centos-node3`上，因此我们在`centos-node3`上执行

```sh
$ $HADOOP_HOME/sbin/yarn-daemon.sh start resourcemanager
```

### 启动 HistoryServer

规划在`centos-node3`上，因此我们在`centos-node3`上执行

```sh
$ $HADOOP_HOME/sbin/mr-jobhistory-daemon.sh start historyserver
```

ps: Hadoop 3.3.0 版本时用这个命令启动`mapred --daemon start`

### 查看HDFS Web页面

位于`centos-node1`的`50070`端口:http://centos-node1:50070/

![image-20200913152341967](https://gitee.com/littlefxc/oss/raw/master/images/image-20200913152341967.png)

### 查看YARN Web 页面

位于`centos-node3`的`8088`端口:http://centos-node3:8088/

![image-20200913152504612](https://gitee.com/littlefxc/oss/raw/master/images/image-20200913152504612.png)

### 查看历史WEB页面

位于`centos-node3`的`19888`端口:http://centos-node3:19888/

![image-20200913152549303](https://gitee.com/littlefxc/oss/raw/master/images/image-20200913152549303.png)

# 测试

为了测试我们使用 `wordcount` 来测试

1. 新建文件

    ```sh
    $ sudo vi /opt/word.txt
    ```

2. 文本内容

    ```sh
    hadoop mapreduce hive
    hbase spark storm
    sqoop hadoop hive
    spark hadoop
    ```

3. 新建`hadoop`里文件夹`demo`

    ```sh
    $ hadoop fs -mkdir /demo
    ```

4. 文件写入

    ```sh
    $ hdfs dfs -put /opt/word.txt /demo/word.txt
    ```

5. 执行输入到`hadoop`的`/output`

    ```sh
    $ yarn jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.4.jar wordcount /demo/word.txt /output
    ```

6. 查看文件列表

    ```sh
    $ hdfs dfs -ls /output
    ```

    ```sh
    Found 2 items
    -rw-r--r--   3 root supergroup          0 2020-09-13 16:01 /output/_SUCCESS
    -rw-r--r--   3 root supergroup         60 2020-09-13 16:01 /output/part-r-00000
    ```

7. 查看文件中内容

    ```sh
    $ hdfs dfs -cat /output/part-r-00000
    ```

    ```sh
    hadoop  3
    hbase   1
    hive    2
    mapreduce       1
    spark   2
    sqoop   1
    storm   1
    ```
