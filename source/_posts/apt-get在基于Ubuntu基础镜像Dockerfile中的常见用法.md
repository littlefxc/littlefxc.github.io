---
title: apt-get在基于Ubuntu基础镜像Dockerfile中的常见用法
date: 2020-05-28 15:39:46
tags: ubuntu,docker,apt-get,
categories: docker
---

首先，在Ubuntu的Docker官方镜像中是没有缓存Apt的软件包列表的。因此在做其他任何基础软件的安装前，都需要至少先做一次apt-get update。有时为了加快apt-get安装软件的速度，还需要修改Apt源的列表文件/etc/apt/sources.list。相应的操作用命令表示如下：

```sh
sed -i s@/archive.ubuntu.com/@/mirrors.aliyun.com/@g /etc/apt/sources.list
```

在容器构建时，为了避免使用apt-get install安装基础软件的过程中需要进行交互操作，使用-y参数来避免安装非必须的文件，从而减小镜像的体积。

```sh
apt-get -y --no-install-recommends install
```

使用`apt-get autoremove`命令移除为了满足包依赖而安装的、但不再需要的包；使用`apt-get clean`命令清除所获得包文件的本地仓库。

`DEBIAN_FRONTEND`这个环境变量，告知操作系统应该从哪儿获得用户输入。如果设置为`noninteractive`，你就可以直接运行命令，而无需向用户请求输入（所有操作都是非交互式的）。这在运行apt-get命令的时候格外有用，因为它会不停的提示用户进行到了哪步并且需要不断确认。非交互模式会选择默认的选项并以最快的速度完成构建。请确保只在Dockerfile中调用的RUN命令中设置了该选项，而不是使用ENV命令进行全局的设置。因为ENV命令在整个容器运行过程中都会生效，所以当你通过BASH和容器进行交互时，如果进行了全局设置那就会出问题。

```Dockerfile
# 正确的做法 - 只为这个命令设置ENV变量
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y python3
# 错误地做法 - 为接下来的任何命令都设置ENV变量，包括正在运行地容器
ENV DEBIAN_FRONTEND noninteractive
RUN apt-get install -y python3
```

示例:

```Dockerfile
FROM ubuntu:16.04

# Ali apt-get source.list
RUN mv /etc/apt/sources.list /etc/apt/sources.list.bak && \
    echo "deb-src http://archive.ubuntu.com/ubuntu xenial main restricted" >/etc/apt/sources.list && \
    echo "deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted" >>/etc/apt/sources.list && \
    echo "deb-src http://mirrors.aliyun.com/ubuntu/ xenial main restricted multiverse universe" >>/etc/apt/sources.list && \
    echo "deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted" >>/etc/apt/sources.list && \
    echo "deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted multiverse universe" >>/etc/apt/sources.list && \
    echo "deb http://mirrors.aliyun.com/ubuntu/ xenial universe" >>/etc/apt/sources.list && \
    echo "deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe" >>/etc/apt/sources.list && \
    echo "deb http://mirrors.aliyun.com/ubuntu/ xenial multiverse" >>/etc/apt/sources.list && \
    echo "deb http://mirrors.aliyun.com/ubuntu/ xenial-updates multiverse" >>/etc/apt/sources.list && \
    echo "deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse" >>/etc/apt/sources.list && \
    echo "deb-src http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse" >>/etc/apt/sources.list && \
    echo "deb http://archive.canonical.com/ubuntu xenial partner" >>/etc/apt/sources.list && \
    echo "deb-src http://archive.canonical.com/ubuntu xenial partner" >>/etc/apt/sources.list && \
    echo "deb http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted" >>/etc/apt/sources.list && \
    echo "deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted multiverse universe" >>/etc/apt/sources.list && \
    echo "deb http://mirrors.aliyun.com/ubuntu/ xenial-security universe" >>/etc/apt/sources.list && \
    echo "deb http://mirrors.aliyun.com/ubuntu/ xenial-security multiverse" >>/etc/apt/sources.list

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        vim \
        python \
        libopencv-dev \
        python-pip \
    && rm -rf /var/lib/apt/lists/*

RUN pip install --upgrade pip \
    numpy \
    pymongo \
    opencv-python
```

## 参考资源

[Docker - 更换内部Ubuntu apt 为国内源](https://www.aiuai.cn/aifarm299.html)