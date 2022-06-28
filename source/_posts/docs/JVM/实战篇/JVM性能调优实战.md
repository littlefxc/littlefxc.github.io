---
title: JVM性能调优实战
date: 2022-06-23 14:47:17
categories: Java
tags:
- jvm
---

# 1 CPU过高问题定位

## 1.1 top + jstack

`top`命令可以显示当前系统正在执行的进程的相关信息，包括进程 ID、内存占用率、CPU 占用率等。

定位的步骤如下：

- 使用 top 命令查看当前 CPU 占用率最高的进程，并取得进程ID；

   ![image-20220627112302300](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220627112302300.png)

- 使用 `top -Hp 进程号` 查看进程中的线程的运行信息，获取线程ID;

- 使用 `printf %x 线程ID` 将线程ID 转换成 16 进制的线程ID；

- 使用`jstack 进程号 > dump.txt`  dump 一下CPU过高的进程；
- 使用`cat dump.txt |grep -A 30 16进制的线程ID`查看dump日志下中的线程详细信息。
