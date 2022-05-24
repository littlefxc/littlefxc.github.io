---
title: Redission 介绍
date: 2021-04-28 10:27:15
tags: 
- redission
- redis
- 分布式锁
---

# Redisson概述

在前面的章节中，我们已经接触了Redis，也知道了如何在Java中调用Redis。Redis有很多Java客户端，我们比较常用有Jedis，spring-data-redis，lettuce等。今天我们给大家介绍另外一个非常好用的Redis的Java客户端——Redisson。我们先看一下Redis官网中介绍的Java客户端列表：
![图片描述](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/5df98c910958344319200842.png)在这个列表中，我们可以看到Redisson的后面有笑脸，有星，说明还是比较受欢迎的。再看看后面的简介，Redisson是一个在Redis服务之上的，分布式、可扩展的Java数据结构。我们进入到Redisson的官网，看看官网是怎么介绍的。

> Redisson是一个在Redis的基础上实现的Java驻内存数据网格（In-Memory Data Grid）。充分的利用了Redis键值数据库提供的一系列优势，基于Java实用工具包中常用接口，为使用者提供了一系列具有分布式特性的常用工具类。使得原本作为协调单机多线程并发程序的工具包获得了协调分布式多机多线程并发系统的能力，大大降低了设计和研发大规模分布式系统的难度。同时结合各富特色的分布式服务，更进一步简化了分布式环境中程序相互之间的协作。它不仅提供了一系列的分布式的Java常用对象，还提供了许多分布式服务。Redisson提供了使用Redis的最简单和最便捷的方法。Redisson的宗旨是促进使用者对Redis的关注分离，从而让使用者能够将精力更集中地放在处理业务逻辑上。

上面一段话看起来有点晦涩难懂，总结起来可以归结为一下几点：

- Redisson提供了使用Redis的最简单和最便捷的方法；
- 开发人员不需过分关注Redis，集中精力关注业务即可；
- 基于Redis，提供了在Java中具有分布式特性的工具类；
- 使Java中的并发工具包获得了协调多机多线程并发的能力；

## Redisson特性

上面我们对Redisson有了一个整体的印象，接下来我们看看它都有哪些特点。

#### 支持的Redis的配置

Redisson支持多种Redis配置，无论你的Redis是单点、集群、主从还是哨兵模式，它都是支持的。只需要在Redisson的配置文件中，增加相应的配置就可以了。

#### 支持的Java实体

Redisson支持多种Java实体，使其具有分布式的特性。我们比较常用的有：AtomicLong（原子Long）、AtomicDouble（原子Double）、PublishSubscribe（发布订阅）等。

#### Java分布式锁与同步器

Redisson支持Java并发包中的多种锁，比如：Lock（可重入锁）、FairLock（公平锁）、MultiLock（联锁）、RedLock（红锁）、ReadWriteLock（读写锁）、Semaphore（信号量）、CountDownLatch（闭锁）等。我们注意到这些都是Java并发包中的类，Redisson借助于Redis又重新实现了一套，使其具有了分布式的特性。以后我们在使用Redisson中的这些类的时候，可以跨进程跨JVM去使用。

#### 分布式Java集合

Redisson对Java的集合类也进行了封装，使其具有分布式的特性。如：Map、Set、List、Queue、Deque、BlockingQueue等。以后我们就可以在分布式的环境中使用这些集合了。

#### 与Spring框架的整合

Redisson可以与Spring大家族中的很多框架进行整合，其中包括：Spring基础框架、Spring Cache、Spring Session、Spring Data Redis、Spring Boot等。在项目中我们可以轻松的与这些框架整合，通过简单的配置就可以实现项目的需求。

## 总结

通过上面的介绍，我相信大家对Redisson有了比较初步的了解，大部分人可能对Redisson有了浓厚的兴趣，迫不及待的想在自己的项目使用Redisson。Redisson的使用方法以及Redisson的分布式锁，我们会在后面的视频教程中给大家做详细的介绍。