---
title: CentOS7下yum安装和配置NFS
date: 2020-04-29 14:54:31
tags: [nfs, centos]
categories: linux
---

NFS 是 Network File System 的缩写，即网络文件系统。功能是让客户端通过网络访问不同主机上磁盘里的数据，主要用在类Unix系统上实现文件共享的一种方法。 本例演示 CentOS 7 下安装和配置 NFS 的基本步骤。

根据官网说明 [Chapter 8. Network File System (NFS) - Red Hat Customer Portal，CentOS 7.4](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/storage_administration_guide/ch-nfs) 以后，支持 NFS v4.2 不需要 rpcbind 了，但是如果客户端只支持 NFC v3 则需要 rpcbind 这个服务。

<!-- more -->

本例演示环境：

| hostname          | IP Address         | 描述 |
| ----------------- | ------------------ | ----- |
| k8s-master-1      | 192.168.200.19     | 服务端 |
| ip-192-168-154-14 | 192.168.154.14     | 客户端 |
| ip-192-168-154-15 | 192.168.154.15     | 客户端 |
| ip-192-168-154-16 | 192.168.154.16     | 客户端 |

`sudo yum install nfs-utils`, 只安装 nfs-utils 即可，rpcbind 属于它的依赖，也会安装上。

## 设置 NFS 服务开机启动

```sh
$ sudo systemctl enable rpcbind
$ sudo systemctl enable nfs
```

## 启动 NFS 服务

```sh
$ sudo systemctl start rpcbind
$ sudo systemctl start nfs
```

## 在各个服务器上关闭防火墙

```sh
$ systemctl stop firewalld.service            #停止firewall
$ systemctl disable firewalld.service         #禁止firewall开机启动
$ systemctl stop iptables.service             #停止iptables
$ systemctl disable iptables.service          #禁止iptables开机启动l
```

## 配置共享目录

服务启动之后，我们在服务端配置一个共享目录

```sh
$ sudo mkdir /home/k8s-projects/k8s-nfs
$ sudo chmod 755 /home/k8s-projects/k8s-nfs
```

根据这个目录，相应配置导出目录

```sh
$ sudo vi /etc/exports
```

添加如下配置

```
/home/k8s-projects/k8s-nfs *(rw,sync,no_root_squash,no_all_squash)
```

1. `/data`: 共享目录位置。
2. `192.168.0.0/24`: 客户端 IP 范围，`*` 代表所有，即没有限制。
3. `rw`: 权限设置，可读可写。
4. `sync`: 同步共享目录。
5. `no_root_squash`: 可以使用 root 授权。
6. `no_all_squash`: 可以使用普通用户授权。

`:wq` 保存设置之后，重启 NFS 服务。

```bash
$ sudo exportfs -r //让上面的配置生效或者重启服务
$ sudo systemctl restart nfs
```

可以检查一下本地的共享目录

```bash
$ showmount -e localhost
Export list for localhost:
/home/k8s-projects/k8s-nfs *
```

这样，服务端就配置好了，接下来配置客户端，连接服务端，使用共享目录。

## Linux 客户端安装

与服务端类似

```bash
$ sudo yum install nfs-utils
```

## Linux 客户端配置

设置 rpcbind 服务的开机启动

```bash
$ sudo systemctl enable rpcbind
```

启动 NFS 服务

```bash
$ sudo systemctl start rpcbind
```

*`注意`*

> 客户端不需要打开防火墙，因为客户端时发出请求方，网络能连接到服务端即可。客户端也不需要开启 NFS 服务，因为不共享目录。

## 客户端连接 NFS

挂载

```bash
$ sudo mount -t nfs 192.168.200.19:/home/k8s-projects/k8s-nfs /mnt/k8s-nf
```

挂载之后，可以使用 `mount` 命令查看一下

```bash
$ showmount -e 192.168.200.19
Export list for 192.168.200.19:
/home/k8s-projects/k8s-nfs *
/root/data/nfs-pic
```

这说明已经挂载成功了。
