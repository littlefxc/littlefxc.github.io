---
title: 编译JDK
date: 2021-07-06 11:06:55
categories: Java
tags:
- jvm
---

# 1 macOS 编译 OpenJDK

目标：编译 OpenJDK17

## 1.1 准备编译环境

1. 首先去应用商店安装 `xcode.app`
2. 安装 `JDK16`（比要编译的JDK低一个版本，如要编译的openjdk17,那就安装jdk16）
3. `brew install freetype ccache`

<!-- more -->

## 1.2 开始编译

### 1.2.1 bash configure

```shell
bash configure --with-debug-level=slowdebug --enable-dtrace --with-jvm-variants=server --with-target-bits=64 --with-num-cores=8 --with-memory-size=8000 --disable-warnings-as-errors
```

参数说明：

- --with-debug-level=slowdebug 启用slowdebug级别调试
- --enable-dtrace 启用dtrace
- --with-jvm-variants=server 编译server类型JVM
- --with-target-bits=64 指定JVM为64位
- --enable-ccache 启用ccache，加快编译（因为在安装 ccache 的时候，总是失败，所以就没有带上这个参数）
- --with-num-cores=8 编译使用CPU核心数
- --with-memory-size=8000 编译使用内存
- --disable-warnings-as-errors 忽略警告

结果如下：

![image-20210706183317055](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210706183317055.png)

### 1.2.2 make images

```shell
make images
```

结果如下：

![image-20210706190102890](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210706190102890.png)

1.2.3 验证

```shell
./build/*/images/jdk/bin/java -version
```

结果如下：

![image-20210706190321790](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210706190321790.png)

## 1.3 FAQ

1. 没有安装 `xcode.app`

![img](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/1460000020736821.png)

解决办法：去应用商店安装

2. 如何完全移除编译包

   ```shell
   make print-configuration > current-configuration
   make dist-clean
   bash configure $(cat current-configuration)
   make
   ```

   

3. 

