---
title: Redis的线程模型
date: 2021-01-26 20:56:29
categories: 缓存
tags:
- redis
---

![https://gitee.com/littlefxc/oss/raw/master/images/redis_thread_model1.png](https://gitee.com/littlefxc/oss/raw/master/images/redis_thread_model1.png)

## **Redis线程模型的组成**

------

1. *多个socket*
2. *IO多路复用程序*
3. *scocket队列*
4. *文件事件分配器*
5. *事件处理器（连接应答处理器，命令请求处理器，命令回复处理器）*

多个 socket 可能会并发产生不同的操作，每个操作对应不同的文件事件，但是 IO 多路复用程序会监听多个 socket，会将 socket 产生的事件放入队列中排队，事件分派器每次从队列中取出一个事件，把该事件交给对应的事件处理器进行处理。

# 流程及原理

------

![redis_thread_model](https://gitee.com/littlefxc/oss/raw/master/images/redis_thread_model.png)

1. 客户端socket01请求redis的server scoket建立连接，此时server socket生成`AE_READABLE`事件，[IO多路复用程序](https://www.notion.so/IO-3896961f1af045ba8a7d2f184f87748d)监听到server socket产生的事件，并将该事件压入队列。

   文件事件分派器从队列中拉取事件交给连接应答处理器，处理器同时生成一个与客户端通信的socket01,并将该scoket01的AE_READABLE事件与命令请求处理器关联

2. 此时客户端scoket01发送一个"set key value“的请求，redis的scoket01接收到AE_READABLE事件，IO多路复用程序监听到事件，将事件压入队列，文件分派器取到事件，由于scoket01已经

   和命令请求处理器关联，所以命令请求处理器开始"set key value",完毕后会将redis的scoket01的`AE_WRITABLE`事件关联到命令回复处理器

3. 如果此时客户端准备好接收返回结果了，向redis中的socket01发起询问请求，那么 redis 中的 socket01 会产生一个 `AE_WRITABLE` 事件，同样压入队列中，事件分派器找到相关联的命令回复处理器，由命令回复处理器对 socket01 输入本次操作的一个结果，比如 `ok`，之后解除 socket01 的 `AE_WRITABLE` 事件与命令回复处理器的关联。

这样便完成了redis的一次通信。

# 文件事件分配器

------

1. Redis 基于 Reactor 模式开发了自己的网络事件处理器： 这个处理器被称为文件事件处理器（file event handler）
2. 文件事件处理器使用 I/O 多路复用（multiplexing）程序来同时监听多个套接字， 并根据套接字目前执行的任务来为套接字关联不同的事件处理器。
3. 当被监听的套接字准备好执行连接应答（accept）、读取（read）、写入（write）、关闭（close）等操作时， 与操作相对应的文件事件就会产生， 这时文件事件处理器就会调用套接字之前关联好的事件处理器来处理这些事件。
4. 文件事件处理器以单线程方式运行， 但通过使用 I/O 多路复用程序来监听多个套接字， 文件事件处理器既实现了高性能的网络通信模型， 又可以很好地与 redis 服务器中其他同样以单线程方式运行的模块进行对接， 这保持了 Redis 内部单线程设计的简单性

# 为什么redis使用单线程模型还能保证高性能？

------

## 纯内存访问

redis 将所有数据放在内存中，内存的响应时长大约为 100 纳秒，这是 redis 的 QPS 过万的重要基础。

## 非阻塞式IO

- 什么是阻塞式 IO

  当我们调用 Scoket 的读写方法，默认它们是阻塞的。

  read() 方法要传递进去一个参数 n，表示读取这么多字节后再返回，如果没有读够 n 字节线程就会阻塞，直到新的数据到来或者连接关闭了， read 方法才可以返回，线程才能继续处理。

  write() 方法会首先把数据写到系统内核为 Scoket 分配的写缓冲区中，当写缓存区满溢，即写缓存区中的数据还没有写入到磁盘，就有新的数据要写道写缓存区时，write() 方法就会阻塞，直到写缓存区中有空闲空间。

- 什么是非阻塞式 IO

  非阻塞 IO 在 Scoket 对象上提供了一个选项`Non_Blocking` ，当这个选项打开时，读写方法不会阻塞，而是能读多少读多少，能写多少写多少。

  能读多少取决于内核为 Scoket 分配的读缓冲区的大小，能写多少取决于内核为 Scoket 分配的写缓冲区的剩余空间大小。读方法和写方法都会通过返回值来告知程序实际读写了多少字节数据。

  有了非阻塞 IO 意味着线程在读写 IO 时可以不必再阻塞了，读写可以瞬间完成然后线程可以继续干别的事了。

## IO多路复用技术

[IO多路复用技术](https://www.notion.so/IO-3896961f1af045ba8a7d2f184f87748d)

- 白话举例:模拟一个tcp服务器处理30个客户socket

  假设你是一个老师，让30个学生解答一道题目，然后检查学生做的是否正确，你有下面几个选择：

  1. 第一种选择：按顺序逐个检查，先检查A，然后是B，之后是C、D。。。这中间如果有一个学生卡主，全班都会被耽误。这种模式就好比，你用循环挨个处理socket，根本不具有并发能力。
  2. 第二种选择：你创建30个分身，每个分身检查一个学生的答案是否正确。 这种类似于为每一个用户创建一个进程或者线程处理连接。
  3. 第三种选择，你站在讲台上等，谁解答完谁举手。这时C、D举手，表示他们解答问题完毕，你下去依次检查C、D的答案，然后继续回到讲台上等。此时E、A又举手，然后去处理E和A。。。 这种就是IO复用模型（Linux下的select、poll和epoll就是干这个的。将用户socket对应的fd注册进epoll，然后epoll帮你监听哪些socket上有消息到达，这样就避免了大量的无用操作。此时的socket应该采用非阻塞模式。这样，整个过程只在调用select、poll、epoll这些调用的时候才会阻塞，收发客户消息是不会阻塞的，整个进程或者线程就被充分利用起来，这就是事件驱动，所谓的reactor模式。）

## 单线程避免了线程切换和竞态产生的消耗

单线程能带来几个好处：

- 第一，单线程可以简化数据结构和算法的实现。并发数据结构实现不但困难而且开发测试比较麻烦
- 第二，单线程避免了线程切换和竞态产生的消耗，对于服务端开发来说，锁和线程切换通常是性能杀手。

**单线程的问题**：对于每个命令的执行时间是有要求的。如果某个命令执行过长，会造成其他命令的阻塞，所以 redis 适用于那些需要快速执行的场景。

# 参考资源

------

[Redis线程模型](https://www.cnblogs.com/volare/p/12283355.html)

[Redis线程模型](https://www.jianshu.com/p/8f2fb61097b8)

[Redis 线程模型_好记性不如烂笔头-CSDN博客](https://blog.csdn.net/m0_37524661/article/details/87086267)

[Redis 单线程模型介绍](https://cloud.tencent.com/developer/article/1403767)

[了解redis的单线程模型工作原理？一篇文章就够了](https://www.cnblogs.com/lm970585581/p/13220299.html)