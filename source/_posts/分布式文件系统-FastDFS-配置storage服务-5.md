---
title: 分布式文件系统-FastDFS-配置storage服务(5)
date: 2021-01-20 15:16:27
categories: 分布式系统
tags:
- 分布式文件系统
---

# 修改storage配置文件

- 修改该storage.cond配置文件

  ![https://climg.mukewang.com/5e0ef5c108c15a6116000535.jpg](https://gitee.com/littlefxc/oss/raw/master/images/5e0ef5c108c15a6116000535.jpg)

```jsx
# 修改组名
group_name=imooc
# 修改storage的工作空间
base_path=/usr/local/fastdfs/storage
# 修改storage的存储空间
store_path0=/usr/local/fastdfs/storage
# 修改tracker的地址和端口号，用于心跳
tracker_server=192.168.1.153:22122

# 后续结合nginx的一个对外服务端口号
http.server_port=8888
```

- 创建目录

```jsx
mkdir /usr/local/fastdfs/storage -p
```

# 启动storage

**前提：必须首先启动tracker**

```jsx
/usr/bin/fdfs_storaged /etc/fdfs/storage.conf
```

检查进程如下：

```jsx
ps -ef|grep storage
```

![https://climg.mukewang.com/5e0ef61c08e61f1d16000147.jpg](https://gitee.com/littlefxc/oss/raw/master/images/5e0ef61c08e61f1d16000147.jpg)

# 测试上传

- 修改的client.conf配置文件

  ![https://climg.mukewang.com/5e0ef62b089e938e15020712.jpg](https://gitee.com/littlefxc/oss/raw/master/images/5e0ef62b089e938e15020712.jpg)

  ```
    base_path=/usr/local/fastdfs/client
    tracker_server=192.168.1.153:22122
  ```

  ```
    mkdir /usr/local/fastdfs/client
  ```

- 测试：

```jsx
wget <https://www.imooc.com/static/img/index/logo.png>
./fdfs_test /etc/fdfs/client.conf upload /home/logo.png
```

上传成功：

![https://climg.mukewang.com/5e0ef64008dfab1e16000857.jpg](https://gitee.com/littlefxc/oss/raw/master/images/5e0ef64008dfab1e16000857.jpg)