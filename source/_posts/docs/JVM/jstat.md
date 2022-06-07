---
title: 内置监控工具 - jstat
date: 2021-08-05 09:52:54
categories: Java
tags:
- jvm
---

# 内置监控工具 - jstat

## 作用
jstat全称JVM Statistics Monitoring Tool，用于监控JVM的各种运行状态。


> TIPS
> 此命令是实验性的，不受支持。

> 参考文档：
> Java 8：https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html
Java 11：https://docs.oracle.com/en/java/javase/11/tools/jstat.html

## 使用说明
命令如下：
```sh
➜ jstat -h
Usage: jstat --help|-options
       jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]


Definitions:
  <option>      指定参数，取值可用jstat -options查看
  <vmid>        VM标识，格式为<lvmid>[@<hostname>[:<port>]]
                <lvmid>：如果lvmid是本地VM，那么用进程号即可; 
                <hostname>：目标JVM的主机名;
                <port>：目标主机的rmiregistry端口;
  -t            用来展示每次采样花费的时间
  <lines>       每抽样几次就列一个标题，默认0，显示数据第一行的列标题
  <interval>    抽样的周期，格式使用:<n>["ms"|"s"]，n是数字，ms/s是时间单位，默认是ms
  <count>       采样多少次停止
  -J<flag>      将<flag>传给运行时系统，例如：-J-Xms48m
  -? -h --help  Prints this help message.
  -help         Prints this help message.
```

option取值如下：
- class：显示类加载器的统计信息
- compiler：显示有关Java HotSpot VM即时编译器行为的统计信息
- gc：显示有关垃圾收集堆行为的统计信息
- gccapacity：统计各个分代（新生代，老年代，持久代）的容量情况
- gccause：显示引起垃圾收集事件的原因
- gcnew：显示有关新生代行为的统计信息
- gcnewcapacity：显示新生代容量
- gcold：显示老年代、元空间区的统计信息
- gcoldcapacity：显示老年代的容量
- gcmetacapacity：显示元空间的容量
- gcutil：显示有关垃圾收集统计信息的摘要
- printcompilation：显示Java HotSpot VM编译方法统计信息

## 输出信息

class:
- Loaded：当前加载的类的数量
- Bytes：当前加载的空间(单位KB)
- Unloaded：卸载的类的数量Number of classes unloaded.
- Bytes：当前卸载的空间(单位KB)
- Time：执行类加载/卸载操作所花费的时间
- compiler：

compiler:

- Compiled：执行了多少次编译任务
- Failed：多少次编译任务执行失败
- Invalid：无效的编译任务数
- Time：执行编译任务所花费的时间
- FailedType：上次失败的编译的编译类型
- FailedMethod：上次失败的编译的类名和方法

gc：

- S0C：第一个存活区(S0)的容量（KB）
- S1C：第二个存活区(S1)的容量（KB）
- S0U：第一个存活区(S0)使用的大小（KB）
- S1U：第二个存活区(S1)使用的大小（KB）
- EC：伊甸园空间容量（KB）
- EU：伊甸园使用的大小（KB）
- OC：老年代容量（KB）
- OU：老年代使用的大小（KB）
- MC：元空间的大小（KB）
- MU：元空间使用的大小（KB）
- CCSC：压缩的类空间大小（KB）
- CCSU：压缩类空间使用的大小（KB）
- YGC：年轻代垃圾收集事件的数量
- YGCT：年轻代垃圾回收时间
- FGC：Full GC事件的数量
- FGCT：Full GC回收时间
- GCT：垃圾收集总时间

gccapacity：

- NGCMN：最小新生代容量（KB）
- NGCMX：最大新生代容量（KB）
- NGC：当前的新生代容量（KB）
- S0C：第一个存活区(S0)的当前容量（KB）
- S1C：第二个存活区(S1)的当前容量（KB）
- EC：当前伊甸园容量（KB）
- OGCMN：最小老年代容量（KB）
- OGCMX：最大老年代容量（KB）
- OGC：当前老年代容量（KB）
- OC：当前old space容量（KB）
- MCMN：最小元空间容量（KB）
- MCMX：最大元空间容量（KB）
- MC：当前元空间的容量（KB）
- CCSMN：压缩的类空间最小容量（KB）
- CCSMX：压缩的类空间最大容量（KB）
- CCSC：当前压缩的类空间大小（KB）
- YGC：年轻代GC事件的数量
- FGC：Full GC事件的数量

gccause：其他展示列和-gcutil一致

- LGCC：导致GC的原因
- GCC：导致当前GC的原因

gcnew：

- S0C：第一个存活区(S0)的容量（KB）
- S1C：第二个存活区(S1)的容量（KB）
- S0U：第一个存活区(S0)的利用率（KB）
- S1U：第二个存活区(S1)的利用率（KB）
- TT：老年代阈值
- MTT：最大老年代阈值
- DSS：期望的存活区大小（KB）
- EC：当前伊甸园容量（KB）
- EU：伊甸园利用率（KB）
- YGC：年轻代GC事件的数量
- YGCT：年轻代垃圾回收时间

gcnewcapacity：

- NGCMN：最小年轻代容量（KB）
- NGCMX：最大年轻代容量（KB）
- NGC：当前年轻代容量（KB） 
- S0CMX：最大S0容量（KB） 
- S0C：当前S0容量（KB） 
- S1CMX：最大S1容量（KB） 
- S1C：当前S1容量（KB） 
- ECMX：最大伊甸园容量（KB） 
- EC：当前伊甸园容量（KB） 
- YGC：年轻代GC事件的数量 
- FGC：Full GC事件的数量

gcold：

- MC：当前元空间使用大小（KB）
- MU：元空间利用率（KB）
- CCSC：压缩的类的大小（KB）
- CCSU：使用的压缩类空间（KB）
- OC：当前的老年代空间容量（KB）
- OU：来年代空间利用率（KB）
- YGC：年轻代GC事件的数量
- FGC：Full GC事件的数量
- FGCT：Full GC垃圾收集时间
- GCT：总垃圾收集时间

gcoldcapacity：

- OGCMN：最小老年代容量（KB）
- OGCMX：最大老年代容量（KB）
- OGC：当前老年代容量（KB）
- OC：当前old space容量（KB）
- YGC：年轻代GC事件的数量
- FGC：Full GC事件的数量
- FGCT：Full GC垃圾收集时间
- GCT：总垃圾收集时间

gcmetacapacity：

- MCMN：最小元空间容量（KB）
- MCMX：最大元空间容量（KB）
- MC：元空间大小（KB）
- CCSMN：压缩的类空间最小容量（KB）
- CCSMX：压缩的类空间最大容量（KB）
- YGC：年轻代GC事件的数量
- FGC：Full GC事件的数量
- FGCT：Full GC垃圾收集时间
- GCT：总垃圾收集时间

gcutil：

- S0：第一个存活区(S0)利用率
- S1：第二个存活区(S1)利用率
- E：Eden空间利用率
- O：老年代空间利用率
- M：元空间利用率
- CCS：压缩的类空间利用率
- YGC：年轻代GC事件的数量
- YGCT：年轻代垃圾回收时间
- FGC：Full GC事件的数量
- FGCT：Full GC垃圾收集时间
- GCT：总垃圾收集时间

printcompilation：

- Compiled：由最近编译的方法去执行的编译任务数
- Size：最近编译的方法的字节码的字节数
- Type：最近编译的方法的编译类型。
- Method：标识最近编译的方法的类名和方法名。类名使用 / 代替点 . 作为名称空间分隔符；方法名称是指定类中的方法。这两个字段的格式与HotSpot -XX:+PrintCompilation 选项一致。

## 使用示例
示例1：查看21891这个进程的gc相关信息，每隔250ms采样1次，采样7次

```sh
jstat -gcutil 21891 250 7
```

示例2：显示有关新生代行为的统计信息，重复列标题：

```sh
jstat -gcnew -h3 21891 250
```

示例3：查看remote.domain机器上的40496这个进程有关垃圾收集统计信息的摘要，每隔1秒采样1次：

```sh
jstat -gcutil 40496@remote.domain 1000
```