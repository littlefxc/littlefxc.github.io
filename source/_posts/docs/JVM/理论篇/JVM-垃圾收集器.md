---
title: "JVM 垃圾收集器"
date: 2022-03-11T16:45:21+08:00
categories: Java
tags:
- jvm
---

[TOC]

## 引言

下面这张图是 Java 中比较主流的基于分代收集理论的垃圾收集器，以及它们能够作用的JVM内存区域。

![image-20220311164628901](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220311164628901.png)

## 术语

- STW：全局停顿，Java 代码停止运行，native 代码继续运行，但不能与 JVM 进行交互。

  - STW 原因：多半由于垃圾回收导致；也可能是由 Dump 线程、死锁检查和 Dump 堆等导致的。

  - STW 危害：服务停止、毫无响应；主从切换、危害生产环境。

- 并行收集：指多个垃圾回收线程并行工作，但是收集的过程中，用户线程还是处于等待状态。

- 并发收集：指用户线程与垃圾收集线程同时工作。
- 吞吐量：CPU 用于运行用户代码的时间与 CPU 总消耗时间的对比
  - 公式：运行用户代码时间/(运行用户代码时间+垃圾收集时间)

## 垃圾收集器介绍

### Serial 收集器（新生代）

![image-20220311171643510](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220311171643510.png)

- 最基本的、历史最悠久的收集器
- 算法：复制算法
- 特点：
  - 简单、高效
  - 单线程
  - 垃圾回收过程中 STW

### ParNew收集器（新生代）

![image-20220311173051813](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220311173051813.png)

- Serial 收集器的多线程版，除了使用多线程外，其它的和Serial收集器一样。
- 特点：
  - 多线程
  - 可以设置垃圾收集的线程数（-XX:ParallelGCThreads）

- 使用场景：主要用来和 CMS 收集器配合使用

### Parallel Scavenge收集器（新生代）

![image-20220314152651631](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220314152651631.png)

- 关注的是吞吐量，也叫吞吐量收集器
- 采用的也是复制算法
- 也是并行的多线程收集器，这一点和 ParNew 类似
- 特点：
  - 可以达到一个可控制的吞吐量，有两个 JVM 参数可以配置：
    - -XX:MaxGCPauseMillis:控制最大的垃圾收集停顿时间（尽力）
    - -XX:GCTimeRatio:设置吞吐量的大小，取值 0-100， 系统花费不超过 1/(1+n) 的时间用于垃圾收集
  - 自适应 GC 策略：可用 -XX:+UseAdptiveSizePolicy 打开
    - 打开自适应策略后，无需手动设置新生代的大小（-Xmn）、Eden 与 Survivor 区的比例（-XX:SurvivorRatio）等参数
    - 虚拟机会自动根据系统的运行状况收集性能情况，动态的调整这些参数，从而达到最优的停顿时间以及最高的的吞吐量
- 使用场景：注重吞吐量的场景

### Serial Old 收集器（老年代）

![image-20220314161053707](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220314161053707.png)

- Serial收集器的老年代
- 算法：标记-整理
- 使用场景：
  - 可以和 Serial、ParNew、Parallel Scavenge 这三个新生代的垃圾收集器配合使用
  - CMS 收集器出现故障的时候，会用 Serial Old 作为后备

### Parallel Old 收集器（老年代）

![image-20220314161516210](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220314161516210.png)

- Parallel Scavenge 收集器的老年代版本
- 算法：标记整理
- 特点：只能和 Parallel Scavenge 配合使用
- 使用场景：关注吞吐量的场景

### CMS 收集器（老年代）

全称叫做 Concurrent Mark Sweep

![image-20220314162206027](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220314162206027.png)

- 并发收集器
- 算法：标记-清除
- CMS 收集器执行过程
  - **初始标记**（initial mark）
     - 标记 GC Roots 能直接关联到的对象 
     - Stop The World
  - **并发标记**（concurrent mark）
    - 找出所有 GC Roots 能关联到的对象
    - 并发执行，无 Stop The World
  - 并发预清理（concurrent-preclean）
    - 重新标记那些在并发标记阶段，引用被更新的对象，从而减少后面重新标记阶段的工作量
    - 并发执行，无 Stop The World
    - 可用 -XX:-CMSPrecleaningEnabled 关闭并发预发清理阶段，默认打开。
  - 并发可中止的预清理阶段（concurrent-abort-oreclan）
    - 和并发预清理做的事一样，并发执行，无 StopTheWorld。
    - 当 Eden 的使用量大于 CMSScheduleRemarkEdenSizeThreashold 的阈值（默认 2M）时，才会执行该阶段
    - 主要作用：允许我们能够控制预清理阶段的结束时机。比如扫描多长时间（CMSMaxAbortablePrecleanTime， 默认 5 秒）或者 Eden 区使用占比打到一定阈值（CMSScheduleRemarkEdenPenetration，默认 50%）就结束本阶段
  - **重新标记**
    - 修正并发标记期间，因为用户程序继续运行，导致标记发生变动的那些对象的标记
    - 一般来说，重新标记花费的时间会比初始标记阶段长一点，但比并发标记的时间短
    - 存在 Stop The World
  - **并发清除**
      - 基于标记结果，清除掉要清除前面标记出来的垃圾
      - 并发执行，无 Stop The World
  - **并发重置**
    - 清理本次 CMS GC 的上下文信息，为下一次 GC 做准备
- 优点：
  - Stop The World 的时间比较短，只有初始标记和重新标记阶段存在 Stop The World，其它阶段都是并发执行的
  - 大多数的过程都是并发执行的

- 缺点：
  - CPU 资源比较敏感：并发阶段可能导致应用吞吐量的降低
  - 无法处理浮动垃圾（并发清除阶段时，用户线程生产出来的垃圾，无法在本次收集时间内处理）
  - 不能等老年代几乎满了才开始收集
    - 预留的内存不够 ->  Concurrent Mode Failure -> Serial Old 作为后备
    - CMSInitiatingOccupancyFraction 设置老年代占比达到多少就触发垃圾收集，默认 68%

  - 内存碎片
    - 标记-清除导致碎片的产生
    - UseCMSCompactAtFullCollection：在完成 Full GC 后是否要进行内存碎片整理，默认开启
    - CMSFullGCsBeforeCompaction：进行几次 Full GC 后就进行一次内存碎片整理，默认 0

- 使用场景：
  - 希望系统停顿时间短，响应速度快的场景，比如各种应用程序


### G1 收集器

- Garbge First

- 面向服务端应用的垃圾收集器

- 内存布局
  ![image-20220314171405933](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220314171405933.png)
  
  - 将整个JVM分成若干个大小相等的区域，每个区域叫做 Region
  - 每个 Region的大小可通过 -XX:G1HeapRegionSize 指定 Region 的大小
  - Region 取值范围为 1M～32M，应为 2 的 N 次幂
  - Region 的分类：Eden、survivor、Old、Humongous
  - 在 G1 里面，同一个代里面的对象可能是不连续的
  - Humongous 是用来存储大对象的，某个对象超过了 Region的一半就认为是大对象，如果对象超级大，就放在多个联系的 humongous 里面
  - G1 会将 Humongous 也看作老年代来处理
  
- 设计思想

  - 若干个 Region
  - 跟踪每个 Region 里面的垃圾堆积的价值大小
  - G1 在后台构建一个优先列表，根据允许的收集时间，优先回收价值高的 Region，这样就可以获得更高的回收效率

- 垃圾收集机制

  - Young GC

    - 当所有的Eden都满了的时候，就会触发 Young GC，
    - 所有的 Eden 里面的对象会转移到 Survivor Region 里面去，
    - 而原先 Survivor Region 里面的对象转移到新的Survivor Region中，或者晋升到 Old Region，
    - 最后，空闲Region会被放入空闲列表中，等待下次被使用

  - Mixed GC

    - 当老年代大小占整个堆的百分比达到一定阈值（可用`-XX:InitiatingHeapOccupancyPercent`指定，默认 45%），就触发 Mixed GC，

    - Mixed GC 会回收所有 Young Region，同时回收**部分** Old Region，

    - Mixed GC 执行过程

      ![image-20220316150715171](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220316150715171.png)
      
      - 初始标记，标记 GC Roots 能直接关联到的对象，和 CMS 类似，**存在** Stop The World
      - 并发标记，同 CMS 的并发标记，并发执行，**没有** Stop The World
      - 最终标记，修正在并发标记期间引起的变动，**存在** Stop The World
      - 筛选回收，对各个Region的回收价值和成本进行排序，根据用户所期望的的停顿时间（根据MaxGCPauseMillis来指定）来制定回收计划，并选择一些 Region 回收，回收过程如下：
        - 选择一系列 Region 构成一个回收集
        - 把决定回收的 Region 中的存活对象复制到空的 Region 中
        - 删掉需回收的Region（无内存碎片）
  
  - Full GC
  
    - 复制对象的内存不够，或者无法分配足够的内存（比如，巨型对象没有足够的连续分区分配）时，会触发Full GC，
    - Full GC 模式下，使用 Serial Old 模式，
    - 因此 G1 的优化原则就是尽量减少 Full GC 的发生。
    - 那么，如何减少Full GC呢？
      - 增加预留内存（增大 -XX:G1ReservePercent，默认为堆的 10%）；
      - 更早地回收垃圾（减少 -XX:InitatingHeapOccupancyPercent，老年代达到该阈值就会触发 Mixed GC，默认 45%）；
      - 增加并发阶段使用的线程数（增大 -XX:ConcGCThreads）
  
- 特点：

  - 可以作用在整个堆、
  - 可控的停顿（MaxGCPauseMillis=200）、
  - 无内存碎片

- 使用场景：

  - 占用内存较大的应用（6G 以上）
  - 替换 CMS 垃圾收集器


### G1 收集器 VS CMS收集器

对于JDK 8：都可以用

- 如果内存小于等于 6G，建议用 CMS，如果内存大于6G，考虑使用 G1。

如果 JDK 版本大于 8 以上，则用 G1， 因为从 JDK9 开始，CMS已被废弃了。


## 链接

1. [分代收集理论](https://blog.csdn.net/Little_fxc/article/details/122603669?spm=1001.2014.3001.5502)

   本文所讲述的垃圾收集器都是基于分代收集理论的

2. [JVM内存结构简单分析](https://blog.csdn.net/Little_fxc/article/details/114004380?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522164698905416780269867798%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=164698905416780269867798&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-5-114004380.nonecase&utm_term=JVM&spm=1018.2226.3001.4450)

   垃圾收集器回收的都有哪些JVM内存区域，这些内存区域在JVM中是怎么分布的？分别有哪些数据？

3. [JVM-垃圾回收算法](https://blog.csdn.net/Little_fxc/article/details/123266153?spm=1001.2014.3001.5501)

   本文提到的垃圾收集器所使用的垃圾回收算法的原理

4. [JVM-垃圾回收](https://blog.csdn.net/Little_fxc/article/details/123210008?spm=1001.2014.3001.5501)

   提到的 GC Roots 是怎么回事
