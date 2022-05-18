---
title: "JVM JDK8的JVM参数"
date: 2022-03-16T17:08:55+08:00
categories: Java
tags:
- jvm
---

> 本文参数基于JDK 8整理

| 收集器                | 参数及默认值                               | 备注                                                         |
| :-------------------- | ------------------------------------------ | ------------------------------------------------------------ |
| Serial                | -XX:+UseSerialGC                           | 虚拟机在Client模式下的默认值，开启后，使用 Serial + Serial Old 的组合 |
| ParNew                | -XX:+UseParNewGC                           | 开启后，使用ParNew + Serial Old的组合                        |
|                       | -XX:ParallelGCThreads=n                    | 设置垃圾收集器在并行阶段使用的垃圾收集线程数，当逻辑处理器数量小于8时，n的值与逻辑处理器数量相同；如果逻辑处理器数量大于8个，则n的值大约为逻辑处理器数量的5/8，大多数情况下是这样，除了较大的SPARC系统，其中n的值约为逻辑处理器的5/16。 |
| Parallel Scavenge     | -XX:+UseParallelGC                         | 虚拟机在Server模式下的默认值，开启后，使用 Parallel Scavenge + Serial Old的组合 |
|                       | -XX:MaxGCPauseMillis=n                     | 收集器尽可能保证单次内存回收停顿的时间不超过这个值，但是并不保证不超过该值 |
|                       | -XX:GCTimeRatio=n                          | 设置吞吐量的大小，取值范围0-100，假设 GCTimeRatio 的值为 n，那么系统将花费不超过 1/(1+n) 的时间用于垃圾收集 |
|                       | -XX:+UseAdaptiveSizePolicy                 | 开启后，无需人工指定新生代的大小（-Xmn）、 Eden和Survisor的比例（-XX:SurvivorRatio）以及晋升老年代对象的年龄（-XX:PretenureSizeThreshold）等参数，收集器会根据当前系统的运行情况自动调整 |
| Serial Old            | 无                                         | Serial Old是Serial的老年代版本，主要用于 Client 模式下的老生代收集，同时也是 CMS 在发生 Concurrent Mode Failure时的后备方案 |
| Parallel Old          | -XX:+UseParallelOldGC                      | 开启后，使用Parallel Scavenge + Parallel Old的组合。Parallel Old是Parallel Scavenge的老年代版本，在注重吞吐量和 CPU 资源敏感的场合，可以优先考虑这个组合 |
| CMS                   | -XX:+UseConcMarkSweepGC                    | 开启后，使用ParNew + CMS的组合；Serial Old收集器将作为CMS收集器出现 Concurrent Mode Failure 失败后的后备收集器使用 |
|                       | -XX:CMSInitiatingOccupancyFraction=68      | CMS 收集器在老年代空间被使用多少后触发垃圾收集，默认68%      |
|                       | -XX:+UseCMSCompactAtFullCollection         | 在完成垃圾收集后是否要进行一次内存碎片整理，默认开启         |
|                       | -XX:CMSFullGCsBeforeCompaction=0           | 在进行若干次Full GC后就进行一次内存碎片整理，默认0           |
|                       | -XX:+UseCMSInitiatingOccupancyOnly         | 允许使用占用值作为启动CMS收集器的唯一标准，一般和CMSFullGCsBeforeCompaction配合使用。如果开启，那么当CMSFullGCsBeforeCompaction达到阈值就开始GC，如果关闭，那么JVM仅在第一次使用CMSFullGCsBeforeCompaction的值，后续则自动调整，默认关闭。 |
|                       | -XX:+CMSParallelRemarkEnabled              | 重新标记阶段并行执行，使用此参数可降低标记停顿，默认打开（仅适用于ParNewGC） |
|                       | -XX:+CMSScavengeBeforeRemark               | 开启或关闭在CMS重新标记阶段之前的清除（YGC）尝试。新生代里一部分对象会作为GC Roots，让CMS在重新标记之前，做一次YGC，而YGC能够回收掉新生代里大多数对象，这样就可以减少GC Roots的开销。因此，打开此开关，可在一定程度上降低CMS重新标记阶段的扫描时间，当然，开启此开关后，YGC也会消耗一些时间。PS. 开启此开关并不保证在标记阶段前一定会进行清除操作，生产环境建议开启，默认关闭。 |
| CMS-Precleaning       | -XX:+CMSPrecleaningEnabled                 | 是否启用并发预清理，默认开启                                 |
| CMS-AbortablePreclean | -XX:CMSScheduleRemark EdenSizeThreshold=2M | 如果伊甸园的内存使用超过该值，才可能进入“并发可中止的预清理”这个阶段 |
| CMS-AbortablePreclean | -XX:CMSMaxAbortablePrecleanLoops=0         | “并发可终止的预清理阶段”的循环次数，默认0，表示不做限制      |
| CMS-AbortablePreclean | -XX:+CMSMaxAbortablePrecleanTime=5000      | “并发可终止的预清理”阶段持续的最大时间                       |
|                       | -XX:+CMSClassUnloadingEnabled              | 使用CMS时，是否启用类卸载，默认开启                          |
|                       | -XX:+ExplicitGCInvokesConcurrent           | 显示调用System.gc()会触发Full GC，会有Stop The World，开启此参数后，可让System.gc()触发的垃圾回收变成一次普通的CMS GC。 |
|                       | -XX:+UseG1GC                               | 使用G1收集器                                                 |
|                       | -XX:G1HeapRegionSize=n                     | 设置每个region的大小，该值为2的幂，范围为1MB到32MB，如不指定G1会根据堆的大小自动决定 |
|                       | -XX:MaxGCPauseMillis=200                   | 设置最大停顿时间，默认值为200毫秒。                          |
|                       | -XX:G1NewSizePercent=5                     | 设置年轻代占整个堆的最小百分比，默认值是5，这是个实验参数。需用-XX:+UnlockExperimentalVMOptions解锁试验参数后，才能使用该参数。 |
|                       | -XX:G1MaxNewSizePercent=60                 | 设置年轻代占整个堆的最大百分比，默认值是60，这是个实验参数。需用-XX:+UnlockExperimentalVMOptions解锁试验参数后，才能使用该参数。 |
|                       | -XX:ParallelGCThreads=n                    | 设置垃圾收集器在并行阶段使用的垃圾收集线程数，当逻辑处理器数量小于8时，n的值与逻辑处理器数量相同；如果逻辑处理器数量大于8个，则n的值大约为逻辑处理器数量的5/8，大多数情况下是这样，除了较大的SPARC系统，其中n的值约为逻辑处理器的5/16。 |
|                       | -XX:ConcGCThreads=n                        | 设置垃圾收集器并发阶段使用的线程数量，设置n大约为ParallelGCThreads的1/4。 |
|                       | -XX:InitiatingHeapOccupancyPercent=45      | 老年代大小达到该阈值，就触发Mixed GC，默认值为45。           |
|                       | -XX:G1MixedGCLiveThresholdPercent=85       | Region中的对象，活跃度低于该阈值，才可能被包含在Mixed GC收集周期中，默认值为85，这是个实验参数。需用-XX:+UnlockExperimentalVMOptions解锁试验参数后，才能使用该参数。 |
|                       | -XX:G1HeapWastePercent=5                   | 设置浪费的堆内存百分比，当可回收百分比小于浪费百分比时，JVM就不会启动Mixed GC，从而避免昂贵的GC开销。此参数相当于用来设置允许垃圾对象占用内存的最大百分比。 |
|                       | -XX:G1MixedGCCountTarget=8                 | 设置在标记周期完成之后，最多执行多少次Mixed GC，默认值为8。  |
|                       | -XX:G1OldCSetRegionThresholdPercent=10     | 设置在一次Mixed GC中被收集的老年代的比例上限，默认值是Java堆的10%，这是个实验参数。需用-XX:+UnlockExperimentalVMOptions解锁试验参数后，才能使用该参数。 |
|                       | -XX:G1ReservePercent=10                    | 设置预留空闲内存百分比，虚拟机会保证Java堆有这么多空间可用，从而防止对象晋升时无空间可用而失败，默认值为Java堆的10％。 |
|                       | -XX:-G1PrintHeapRegions                    | 输出Region被分配和回收的信息，默认false                      |
|                       | -XX:-G1PrintRegionLivenessInfo             | 在清理阶段的并发标记环节，输出堆中的所有Regions的活跃度信息，默认false |
| Shenandoah            | -XX:+UseShenandoahGC                       | 使用UseShenandoahGC，这是个实验参数，需用-XX:+UnlockExperimentalVMOptions解锁试验参数后，才能使用该参数；另外该参数只能在Open JDK中使用，Oracle JDK无法使用 |
| ZGC                   | -XX:+UseZGC                                | 使用ZGC，这是个实验参数，需用-XX:+UnlockExperimentalVMOptions解锁试验参数后，才能使用该参数； |
| Epsilon               | -XX:+UseEpsilonGC                          | 使用EpsilonGC，这是个实验参数，需用-XX:+UnlockExperimentalVMOptions解锁试验参数后，才能使用该参数； |

引用：

[Getting Started with the G1 Garbage Collector](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/G1GettingStarted/index.html)

[JVM各个发行版的JVM参数](https://chriswhocodes.com)
