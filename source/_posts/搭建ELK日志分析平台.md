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

    - 手动安装 Logstash

        ```sh

        ```

    - APT 工具安装

        ```sh
        # 下载并安装公共签名密钥：
        wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
        # 可能需要安装 apt-transport-https
        sudo apt-get install apt-transport-https
        # 添加库定义文件到 /etc/apt/sources.list.d/elastic-7.x.list
        echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
        # 更新并安装
        sudo apt-get update && sudo apt-get install logstash
        ```

    - YUM 工具安装

        下载并安装公共签名密钥：

        ```sh
        rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
        ```

        在 `/etc/yum.repos.d/` 文件夹中将下面的内容添加到 `logstash.repo` 文件中

        ```sh
        [logstash-7.x]
        name=Elastic repository for 7.x packages
        baseurl=https://artifacts.elastic.co/packages/7.x/yum
        gpgcheck=1
        gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
        enabled=1
        autorefresh=1
        type=rpm-md
        ```

        安装 Logstash

        ```sh
        sudo yum install logstash
        ```

    - Docker 拉取镜像

        ```sh
        docker pull docker.elastic.co/logstash/logstash:7.1.1
        ```

3. 安装 elasticsearch 集群

