---
title: 搭建ELK日志分析平台
date: 2019-05-23 09:58:07
categories: 日志
tags:
- ELK
---

## 搭建ELK日志分析平台



### Logstash

#### 下载

直接下载官方发布的二进制包的，可以访问 [https://www.elastic.co/downloads/logstash](https://www.elastic.co/downloads/logstash) 页面找对应操作系统和版本，点击下载即可。不过更推荐使用软件仓库完成安装。

#### 安装

1. 确保系统环境中安装了 JDK8 及以上

    在 Linux 中配置 JAVA 环境：

    ```sh
    export JAVA_HOME=/usr/local/java/jdk1.8.0_112
    export PATH=$JAVA_HOME/bin:$PATH
    export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
    ```

2. 安装 Logstash

3. 安装 elasticsearch 集群

