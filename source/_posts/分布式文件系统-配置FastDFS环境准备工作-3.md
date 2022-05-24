---
title: 分布式文件系统-配置FastDFS环境准备工作(3)
date: 2021-01-20 15:11:09
categories: 分布式系统
tags:
- 分布式文件系统
---

# 参考文献

[fastDFS 一二事 - 简易服务器搭建（单linux）](https://www.cnblogs.com/leechenxiang/p/5406548.html)

[腾讯云服务器 安装fastdfs文件服务器](https://www.cnblogs.com/leechenxiang/p/7089778.html)

[happyfish100/fastdfs](https://github.com/happyfish100/fastdfs/wiki)

<!-- more -->

# 环境准备

- Centos7.x 两台，分别安装tracker与storage
- 下载安装包：
  - libfatscommon：FastDFS分离出的一些公用函数包
  - FastDFS：FastDFS本体
  - fastdfs-nginx-module：FastDFS和nginx的关联模块
  - nginx：发布访问服务

# 安装步骤 (tracker与storage都要执行)

1. 安装基础环境

   ```
   yum install -y gcc gcc-c++
   yum -y install libevent
   ```

2. 安装libfatscommon函数库安装的目录从控制台看一下：

   ```
   # 解压
   tar -zxvf libfastcommon-1.0.42.tar.gz
   ```

   - 进入libfastcommon文件夹，编译并且安装

   ```
   ./make.sh
   ./make.sh install
   ```

3. 安装fastdfs主程序文件

   ```
   # 解压
   tar -zxvf fastdfs-6.04.tar.gz
   ```

   进入到fastdfs目录，查看fastdfs安装配置

   ```
   cd fastdfs-6.04/
   vim make.sh
   ```

   ```
   TARGET_PREFIX=$DESTDIR/usr
   TARGET_CONF_PATH=$DESTDIR/etc/fdfs
   TARGET_INIT_PATH=$DESTDIR/etc/init.d
   ```

   安装fastdfs

   ```
   ./make.sh
   ./make.sh install
   ```

   ![https://climg.mukewang.com/5e0ef484082a831c05840194.jpg](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/5e0ef484082a831c05840194.jpg)

   如上图，

   - `/usr/bin` 中包含了可执行文件；

   - `/etc/fdfs` 包含了配置文件；

     ![https://climg.mukewang.com/5e0ef4aa082ddd3610940470.jpg](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/5e0ef4aa082ddd3610940470.jpg)

     ![https://climg.mukewang.com/5e0ef4bc083b7c0109940238.jpg](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/5e0ef4bc083b7c0109940238.jpg)

- 拷贝配置文件如下：

```jsx
cp /home/software/FastDFS/fastdfs-6.04/conf/* /etc/fdfs/
```

![https://climg.mukewang.com/5e0ef4cf08b65f3b13560470.jpg](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/5e0ef4cf08b65f3b13560470.jpg)