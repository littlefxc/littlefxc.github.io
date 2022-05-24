---
title: "JVM 垃圾回收算法"
date: 2022-03-01T17:19:01+08:00
categories: Java
tags:
- jvm
---

## 0 概述

基础垃圾算法：标记-清除、标记-整理、复制

综合垃圾回收算法：分代收集算法、增量算法

## 1 标记-清除（Mark-Sweep）

![](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/20220303214940.png)

1. 标记需要回收的对象

2. 清理掉要回收的对象

缺点：垃圾回收之后会存在内存碎片

## 2 标记-整理（Mark-Compact）

![](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/20220303222629.png)

1. 标记需要回收的对象

2. 把所有的存活对象压缩的内存的一端
3. 清理掉边界外的所有空间

这个算法可以避免标记-清除算法会产生的内存碎片

## 3 复制（Copy）

![image-20220303223424546](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220303223424546.png)

1. 把内存分为两块，每次只使用一块

2. 把当前使用的内存中的存活对象复制到未使用的内存中去，然后清除掉正在使用的内存中的所有对象
3. 交换两个内存的角色，等待下次回收

## 4 三种算法对比

| 回收算法  | 有点           | 缺点                               |
| --------- | -------------- | ---------------------------------- |
| 标记-清除 | 实现简单       | 存在内存碎片、分配内存速度会受影响 |
| 标记-整理 | 无碎片         | 整理内存存在开销                   |
| 复制      | 性能好、无碎片 | 内存利用率低                       |

 ## 5 分代收集算法

1. 各种商业虚拟机堆内存的垃圾回收基本上都采用了垃圾回收

2. 根据对象的存活周期，把内存分成多个区域，不同区域使用不同的回收算法回收对象

   ![image-20220311113729890](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220311113729890.png)

### 5.1 回收类型

- 新生代回收（Minor GC|Young GC）
- 老年代回收（Major GC）
- 清理整个堆的回收（Full GC）

一般的，发生 Major GC 时同时一般也会发生 Minor GC，所以 Major GC 约等于 Full GC。

对象分配过程见[JVM之对象的创建过程](https://blog.csdn.net/Little_fxc/article/details/118633548?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522164697019016780366533422%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=164697019016780366533422&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-2-118633548.nonecase&utm_term=对象&spm=1018.2226.3001.4450)

分代收集算法调优原则：

- 合理设置 survivor 区域的大小，避免内存浪费；
- GC尽量发生在新生代，尽量减少 Full GC 的发生。

相关的JVM参数

![image-20220311162205506](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220311162205506.png)

## 6 增量算法

如果内存非常大，如果一次性回收所有垃圾，势必会消耗很多的时间，从而造成系统长时间停摆。

1. 每次只收集一小片区域的内存空间的垃圾

## 链接

1. [垃圾回收](https://blog.csdn.net/Little_fxc/article/details/123210008?spm=1001.2014.3001.5501)

   什么场景下该使用什么垃圾回收策略？垃圾回收发生在哪些区域？对象在什么时候能够被回收？

2. [JVM分代收集理论](https://blog.csdn.net/Little_fxc/article/details/122603669?spm=1001.2014.3001.5501)

   与章节 5 相关，主要介绍分代收集算法实现的理论依据

3. [JVM-内存结构](https://blog.csdn.net/Little_fxc/article/details/114004380?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522164698742116780269896838%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=164698742116780269896838&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-6-114004380.nonecase&utm_term=JVM&spm=1018.2226.3001.4450)

   主要是基于分代收集理论的JVM内存结构简单分析，包括堆、虚拟机栈、本地方法栈、程序计数器和方法区
