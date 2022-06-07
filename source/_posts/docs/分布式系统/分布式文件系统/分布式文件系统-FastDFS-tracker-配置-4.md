---
title: 分布式文件系统-FastDFS-tracker-配置(4)
date: 2021-01-20 15:14:30
categories: 分布式系统
tags:
- 分布式文件系统
---

# 说明

tracker和storage都是同一个fastdfs的主程序的两个不同概念，配置不同的配置文件就可以设定为tracker或者storage

<!-- more -->

# 配置tracker

`/etc/fdfs`下都是一些配置文件，配置tracker即可

```jsx
vim tracker.conf
```

![https://climg.mukewang.com/5e0ef55e08c034fc11160388.jpg](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/5e0ef55e08c034fc11160388.jpg)

- 修改tracker配置文件，此为tracker的工作目录，保存数据以及日志

  ```
  base_path=/usr/local/fastdfs/tracker
  ```

  ```
  mkdir /usr/local/fastdfs/tracker -p
  ```

# 启动tracker服务

```jsx
/usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf
```

检查进程如下：

```jsx
ps -ef|grep tracker
```

![https://climg.mukewang.com/5e0ef5720852057816000146.jpg](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/5e0ef5720852057816000146.jpg)

- 停止tracker

  ```
  /usr/bin/stop.sh /etc/fdfs/tracker.conf
  ```