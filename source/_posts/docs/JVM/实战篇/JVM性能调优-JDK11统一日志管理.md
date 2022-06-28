---
title: JVM性能调优-JDK11统一日志管理
date: 2022-06-24 11:23:29
categories: Java
tags:
- jvm
---

JDK 11统一日志管理
============

> **TIPS**
>
> 本文基于JDK 11编写，理论适用于JDK 9及更高版本。

可使用-Xlog选项，启用统一日志管理。

Xlog选项支持的参数如下：

* -Xlog：使用info级别启用JVM日志
  
* -Xlog:help：打印Xlog帮助文档
  
* -Xlog:disable：关闭所有日志记录并清除日志记录框架的所有配置，包括警告和错误的默认配置
  
* -Xlog[:option]：按照命令行上出现的顺序应用多个参数。同一输出的多个参数按其给定顺序覆盖。option的格式为：
  
    ```
    [:[what][:[output][:[decorators][:output-options[,...]]]]]
    ```
    
    其中：
    
    * what：指定level和tag的组合，格式：`tag1[+tag2...][*][=level][,...]` 。除非用 `*` 指定了通配符，否则只有匹配了指定tag的日志消息才会被匹配。
      
    * output：设置输出类型。默认为stdout。
      
    * decorators：使用一系列自定义的装饰器去配置output。缺省的装饰器为uptime、level和tags。
      
    * output-options：设置Xlog的日志输出选项，格式：
      
      ```
      filecount=file-count filesize=file size with optional K, M or G suffix
      ```
      

默认配置
----

如果只指定了 `-Xlog` ，则使用默认配置，等价于如下配置：

    -Xlog:all=warning:stdout:uptime,level,tags


在运行时控制日志
--------

可用jcmd的 `VM.log` 诊断命令在运行时控制日志记录。例如：

    jcmd 48758 VM.log output what


-Xlog标签和级别
-----------

每个日志消息都有一个级别和与之关联的tag集合。消息的级别与其详细信息相对应，tag集与消息包含的内容或者消息所涉及的JVM组件（例如GC、编译器或线程）相对应。

可用的日志级别：

* `off`
* `trace`
* `debug`
* `info`
* `warning`
* `error`

可用的日志标签（ 如指定为all，则表示下面所有标签的组合）：

* `add`
* `age`
* `alloc`
* `annotation`
* `aot`
* `arguments`
* `attach`
* `barrier`
* `biasedlocking`
* `blocks`
* `bot`
* `breakpoint`
* `bytecode`
* `census`
* `class`
* `classhisto`
* `cleanup`
* `compaction`
* `comparator`
* `constraints`
* `constantpool`
* `coops`
* `cpu`
* `cset`
* `data`
* `defaultmethods`
* `dump`
* `ergo`
* `event`
* `exceptions`
* `exit`
* `fingerprint`
* `freelist`
* `gc`
* `hashtables`
* `heap`
* `humongous`
* `ihop`
* `iklass`
* `init`
* `itables`
* `jfr`
* `jni`
* `jvmti`
* `liveness`
* `load`
* `loader`
* `logging`
* `mark`
* `marking`
* `metadata`
* `metaspace`
* `method`
* `mmu`
* `modules`
* `monitorinflation`
* `monitormismatch`
* `nmethod`
* `normalize`
* `objecttagging`
* `obsolete`
* `oopmap`
* `os`
* `pagesize`
* `parser`
* `patch`
* `path`
* `phases`
* `plab`
* `preorder`
* `promotion`
* `protectiondomain`
* `purge`
* `redefine`
* `ref`
* `refine`
* `region`
* `remset`
* `resolve`
* `safepoint`
* `scavenge`
* `scrub`
* `setting`
* `stackmap`
* `stacktrace`
* `stackwalk`
* `start`
* `startuptime`
* `state`
* `stats`
* `stringdedup`
* `stringtable`
* `subclass`
* `survivor`
* `sweep`
* `system`
* `task`
* `thread`
* `time`
* `timer`
* `tlab`
* `unload`
* `update`
* `verification`
* `verify`
* `vmoperation`
* `vtables`
* `workgang`

下表描述了标签和级别的组合：

| 日志标签                                    | 描述                                                         |
| :------------------------------------------ | :----------------------------------------------------------- |
| ```-Xlog:gc```                              | 打印 ```gc``` 信息以及垃圾回收发生的时间。                   |
| ```-Xlog:gc*```                             | 打印至少包含 ```gc``` 标签的日志消息。它还可以具有与其关联的其他标签。但是，它不会提供```phase```级别信息。 |
| ```-Xlog:gc*=trace```                       | 打印trace级别及更高的```gc```日志记录信息。输出显示所有```gc```相关标签以及详细的日志记录信息。 |
| ```-Xlog:gc+phases=debug```                 | 打印不同的```phase```级别信息。这提供了在```debug```级别记录的详细信息级别。 |
| ```-Xlog:gc+heap=debug```                   | 在gc之前和之后打印堆的使用详细。这将会以debug级别打印带有tag和heap的标记的日志 |
| ```-Xlog:safepoint```                       | 在同一级别上打印有关应用并发时间（application concurrent time）和停顿时间（application stop time）的详细信息。 |
| ```-Xlog:gc+ergo*=trace```                  | 以```trace``` 级别同时打印```gc```和```ergo```消息的组合。该信息包括有关堆大小和收集集构造的所有详细信息。 |
| ```-Xlog:gc+age=trace```                    | 以trace级别 打印存活区的大小、以及存活对象在存活区的年龄分布 |
| ```-Xlog:gc*:file=::filecount=,filesize=``` | 将输出重定向到文件，在其中指定要使用的文件数和文件大小，单位 ```kb``` |

-Xlog输出
--------

`-Xlog` 支持以下类型的输出：

*   `stdout` ：将输出发送到标准输出
*   `stderr` ：将输出发送到stderr
*   `file=filename` ：将输出发送到文本文件。你还可以让文件按照文件大小轮换，例如每记录10M就轮换，只保留5个文件等。默认情况下，最多保留5个20M的文件。可使用 `filesize=10M, filecount=5` 格式去指定文件大小和保留的文件数。

装饰器
---

装饰器用来装饰消息，记录与消息有关的信息。可以为每个输出配置一组自定义的装饰器，输出顺序和定义的顺序相同。缺省的装饰器为uptime、level和tags。none表示禁用所有的装饰器。

下表展示了所有可用的装饰器：

| 装饰器                         | 描述                                                      |
| :----------------------------- | :-------------------------------------------------------- |
| ```time``` or ```t```          | ISO-8601格式的当前日期时间                                |
| ```utctime``` or ```utc```     | Universal Time Coordinated or Coordinated Universal Time. |
| ```uptime``` or ```u```        | JVM启动了多久，以秒或毫秒为单位。例如6.567s.              |
| ```timemillis``` or ```tm```   | 相当于 ```System.currentTimeMillis()```                   |
| ```uptimemillis``` or ```um``` | JVM启动以来的毫秒数                                       |
| ```timenanos``` or ```tn```    | 相当于 ```System.nanoTime()```                            |
| ```uptimenanos``` or ```un```  | JVM启动以来的纳秒数                                       |
| ```hostname``` or ```hn```     | 主机名                                                    |
| ```pid``` or ```p```           | The process identifier.                                   |
| ```tid``` or ```ti```          | 打印线程号                                                |
| ```level``` or ```l```         | 与日志消息关联的级别                                      |
| `tags` or `tg`                 | 与日志消息关联的标签集                                    |

使用示例
----

    # 示例1：使用info级别记录所有信息到stdout，装饰器使用uptime、level及tags
    # 等价于-Xlog:all=info:stdout:uptime,levels,tags
    -Xlog
    
    # 示例2：以info级别打印使用了gc标签的日志到stdout
    -Xlog:gc
    
    # 示例3：使用默认装饰器，info级别，将使用gc或safepoint标签的消息记录到stdout。
    # 如果某个日志同时标签了gc及safepoint，不会被记录
    -Xlog:gc,safepoint
    
    # 示例4：使用默认装饰器，debug级别，打印同时带有gc和ref标签的日志。
    # 仅使用gc或ref的日志不会被记录
    -Xlog:gc+ref=debug
    
    # 示例5：不使用装饰器，使用debug级别，将带有gc标签的日志记录到gc.txt中
    -Xlog:gc=debug:file=gc.txt:none
    
    # 示例6：以trace级别记录所有带有gc标签的日志到gctrace.txt文件集中，该文件集中的文件最大1M，保留5个文件；使用的装饰器是uptimemillis、pids
    -Xlog:gc=trace:file=gctrace.txt:uptimemillis,pids:filecount=5,filesize=1024
    
    # 示例7：使用trace级别，记录至少带有gc及meta标签的日志到gcmetatrace.txt，同时关闭带有class的日志。某个消息如果同时带有gc、meta及class，将不会被记录，因为class标签被关闭了。
    -Xlog:gc+meta*=trace,class*=off:file=gcmetatrace.txt


旧式GC日志和Xlog的对照
--------------

| 旧式GC标记                               | Xlog配置                          | 注释                                                         |
| :--------------------------------------- | :-------------------------------- | :----------------------------------------------------------- |
| ```G1PrintHeapRegions```                 | ```-Xlog:gc+region=trace```       | -                                                            |
| ```GCLogFileSize```                      | No configuration available        | 日志轮换由框架处理                                           |
| ```NumberOfGCLogFiles```                 | Not Applicable                    | 日志轮换由框架处理                                           |
| ```PrintAdaptiveSizePolicy```            | ```-Xlog:gc+ergo*=level```        | 使用debug级别可打印大部分信息，使用trace级别可打印所有 ```PrintAdaptiveSizePolicy``` 打印的信息 |
| ```PrintGC```                            | ```-Xlog:gc```                    | -                                                            |
| ```PrintGCApplicationConcurrentTime```   | ```-Xlog:safepoint```             | 注意： ```PrintGCApplicationConcurrentTime``` 和 ```PrintGCApplicationStoppedTime```  是记录在同一tag之上的，并且没有被分开 |
| ```PrintGCApplicationStoppedTime```      | ```-Xlog:safepoint```             | 注意： ```PrintGCApplicationConcurrentTime``` 和 ```PrintGCApplicationStoppedTime```  是记录在同一tag之上的，并且没有被分开 |
| ```PrintGCCause```                       | Not Applicable                    | Xlog总是会记录GC cause                                       |
| ```PrintGCDateStamps```                  | Not Applicable                    | 日期戳由框架记录                                             |
| ```PrintGCDetails```                     | ```-Xlog:gc*```                   | -                                                            |
| ```PrintGCID```                          | Not Applicable                    | Xlog总是会记录GC ID                                          |
| ```PrintGCTaskTimeStamps```              | ```-Xlog:gc+task*=debug```        | -                                                            |
| ```PrintGCTimeStamps```                  | Not Applicable                    | 时间戳由框架记录                                             |
| ```PrintHeapAtGC```                      | ```-Xlog:gc+heap=trace```         | -                                                            |
| ```PrintReferenceGC```                   | ```-Xlog:gc+ref*=debug```         | 注意：旧式写法中，```PrintGCDetails```启用时， ```PrintReferenceGC``` 才会生效 |
| ```PrintStringDeduplicationStatistics``` | ```-Xlog:gc+stringdedup*=debug``` | -                                                            |
| ```PrintTenuringDistribution```          | ```-Xlog:gc+age*=level```         | 使用debug日志级别记录最相关信息；trace级别记录所有 ```PrintTenuringDistribution``` 会打印的信息。 |
| ```UseGCLogFileRotation```               | Not Applicable                    | 用来记录 ```PrintTenuringDistribution```                     |

旧式运行时日志和Xlog的对照
---------------

| 旧式运行时标记                  | Xlog配置                                  | 注释                                                         |
| :------------------------------ | :---------------------------------------- | :----------------------------------------------------------- |
| ```TraceExceptions```           | ```-Xlog:exceptions=info```               | -                                                            |
| ```TraceClassLoading```         | ```-Xlog:class+load=level```              | 使用info级别记录常规信息，debug级别记录额外信息。在统一日志记录语法中， ```-verbose:class``` 等价于 ```-Xlog:class+load=info,class+unload=info```. |
| ```TraceClassLoadingPreorder``` | ```-Xlog:class+preorder=debug```          | -                                                            |
| ```TraceClassUnloading```       | ```-Xlog:class+unload=level```            | 使用info级别记录常规信息，debug级别记录额外信息。在统一日志记录语法中， ```-verbose:class``` 等价于 ```-Xlog:class+load=info,class+unload=info```. |
| ```VerboseVerification```       | ```-Xlog:verification=info```             | -                                                            |
| ```TraceClassPaths```           | ```-Xlog:class+path=info```               | -                                                            |
| ```TraceClassResolution```      | ```-Xlog:class+resolve=debug```           | -                                                            |
| ```TraceClassInitialization```  | ```-Xlog:class+init=info```               | -                                                            |
| ```TraceLoaderConstraints```    | ```-Xlog:class+loader+constraints=info``` | -                                                            |
| ```TraceClassLoaderData```      | ```-Xlog:class+loader+data=level```       | 使用info级别记录常规信息，debug级别记录额外信息。            |
| ```TraceSafepointCleanupTime``` | ```-Xlog:safepoint+cleanup=info```        | -                                                            |
| ```TraceSafepoint```            | ```-Xlog:safepoint=debug```               | -                                                            |
| ```TraceMonitorInflation```     | ```-Xlog:monitorinflation=debug```        | -                                                            |
| ```TraceBiasedLocking```        | ```-Xlog:biasedlocking=level```           | 使用info级别记录常规信息，debug级别记录额外信息。            |
| ```TraceRedefineClasses```      | ```-Xlog:redefine+class*=level```         | 使用level=info，level=debug和level=trace提供越来越多的信息。 |

参考文档
----

*   [Enable Logging with the JVM Unified Logging Framework](https://docs.oracle.com/en/java/javase/11/tools/java.html#GUID-BE93ABDC-999C-4CB5-A88B-1994AAAC74D5)
