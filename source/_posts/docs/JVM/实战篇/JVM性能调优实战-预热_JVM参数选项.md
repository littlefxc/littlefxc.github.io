---
title: JVM性能调优实战-JVM参数选项
date: 2022-06-23 14:47:17
categories: Java
tags:
- jvm
---

# 预热：JVM参数选项

> **TIPS**
> 
> 本文基于JDK 11整理，理论兼容JDK 8及更高版本。

本文探讨JVM选项的分类与方式。

## 1 标准选项(Standard Options for Java)

### 1.1 作用

用于执行常见操作，例如检查JRE版本，设置类路径，启用详细输出等，各种虚拟机的实现都会去支持标准参数

### 1.2 查看支持的参数

    java -help

### 1.3 使用格式

格式不统一，以 `java -help` 的结果为准

### 1.4 使用示例

    java -version
    java -agentlib:jdwp=help
    java --show-version


   1.5 常用标准选项

> **TIPS**
> 
> 完整的标准选项详见：
> 
> - JDK 11：[Standard Options for Java](https://docs.oracle.com/en/java/javase/11/tools/java.html#GUID-3B1CE181-CD30-4178-9602-230B800D4FAE__BABDJJFI)
> - JDK 8：[Standard Options](https://docs.oracle.com/javase/8/docs/technotes/tools/windows/java.html#BABDJJFI)

- 
–class-path classpath, -classpath classpath, or -cp classpath

通知JVM类搜索路径。如果指定了`-classpath`，则JVM就忽略`CLASSPATH`中指定的路径。各路径之间以分号隔开。如果`-classpath`和`CLASSPATH`都没有指定，则JVM从当前路径寻找class

- 
-server

以server模式启动JVM，与client情况恰好相反。适合生产环境，适用于服务器。64位的JVM自动以server模式启动

- 
-client

以client模式启动JVM，这种方式启动速度快，但运行时性能和内存管理效率不高，适合客户端程序或者开发调试。64位的JVM不支持client模式

- 
-Dproperty=value

设置系统属性值。其中， `property` 是属性名称， `value` 是属性的值，如果value有空格，则需要使用双引号，例如`-Dfoo="foo bar"`

- 
-javaagent:jarpath[=options]

加载指定的Java编程语言代理

- 
-verbose:class

显示类加载相关的信息，当报找不到类或者类冲突时可用此参数诊断

- 
-verbose:gc

显示垃圾收集事件的相关信息

- 
-verbose:jni

显示本机方法和其他Java本机接口（JNI）的相关信息

- 
–version

展示JDK版本（打印到错误流error stream）

- 
-version

展示JDK版本（打印到输出流out stream）

## 2 附加选项(Extra Options for Java)

> **TIPS**
> 
> - 额外参数(Extra Options for Java)，是JDK 11文档中的说法。JDK 8的文档将额外参数称之为“非标准参数(Non-Standard Options)”，但可以理解为是一个东西，只是改了个名而已。

### 2.1 作用

HotSpot虚拟机的通用选项，其他厂牌的JVM不一定会支持这些选项，并且在未来可能会发生变化。这些选项以-X开头。

### 2.2 查看支持的参数

    java -X

### 2.3 使用格式

大部分附加选项的格式都是-X属性名，例如-Xcomp，但也有例外，比如-Xmx500m。具体可参考 `java -X` 的输出

### 2.4 使用示例

    java -Xmx80m
    java -Xint 

### 2.5 常用附加选项

> **TIPS**
> 
> 完整的附加选项详见：
> 
> - JDK 11：[Extra Options for Java](https://docs.oracle.com/en/java/javase/11/tools/java.html#GUID-3B1CE181-CD30-4178-9602-230B800D4FAE__BABHDABI)
> - JDK 8：[Non-Standard Options](https://docs.oracle.com/javase/8/docs/technotes/tools/windows/java.html#BABHDABI)

- 
-Xcomp

在第一次调用时强制编译方法。默认情况下，client模式下会解释执行1000次（JDK 11下），server模式下会解释执行10000次，并收集信息，此后才可能编译运行。指定该选项将禁用解释方法调用。此外，还可使用 `-XX:CompileThreshold` 选项更改在编译之前解释执行方法的调用次数

- 
-Xint

以解释模式运行

- 
-Xmixed

热点方法以编译模式运行，其他方法以解释模式运行

- 
-Xloggc:option

将GC事件的相关信息记录到文件中

- 
-Xnoclassgc

禁用类的垃圾收集。使用该参数可节省一些GC时间，缩短应用程序运行期间的停顿。但一旦使用该参数，那么应用程序中的类对象就会始终被视为活动对象，从而导致这块内存被永久占用。如果使用不当，将会导致内存溢出（out-of-memory exception）

- 
-Xshare:mode

设置类数据共享（ class data sharing (CDS) ）模式。mode的取值如下：

- auto：尽可能使用CDS，32位Hotspot JVM默认值
- on：开启CDS。如果CDS无法被开启将会打印错误信息。（**注**：此选项只应用于测试目的，并且可能由于操作系统使用地址空间布局随机化而导致间歇性故障，不应在生产环境使用）
- off：不使用CDS

- 
-XshowSettings:category

展示所有设置。category取值如下：

- all：展示所有设置（默认值）
- locale：展示语言环境相关的设置
- properties：展示系统属性相关的设置
- vm：展示JVM的设置
- system：显示Linux主机系统或容器配置

- 
-Xmnsize

设置年轻代的初始值及最大值，以字节为单位。也可在size后追加字母 `k` 或 `K` 表示千字节， `m` 或 `M` 表示兆字节，或 `g` 或 `G` 表示千兆字节，例如-Xmn256m。此外，还可用 `-XX:NewSize` 设置年轻代初始大小， `-XX:MaxNewSize`设置年轻代最大大小

- 
-Xmssize

设置堆内存的初始大小，以字节为单位。此值必须是1024的倍数且大于1MB。如果未设置此选项，则将堆内存初始大小设为为老年代和年轻代分配的大小之和。设置格式同Xmn，例如-Xms6144k。

- 
-Xmxsize

设置堆内存的最大大小，以字节为单位。此值必须是1024的倍数且大于2MB。等效于`-XX:MaxHeapSize` 。

- 
-Xsssize

设置线程栈大小，以字节为单位。默认值取决于平台：

- Linux / x64（64位）：1024 KB
- macOS（64位）：1024 KB
- Oracle Solaris / x64（64位）：1024 KB
- Windows：默认值取决于虚拟内存

## 3 高级选项(Advanced Options)

### 3.1 作用

高级选项是为开发人员提供的选项，用于调整Java HotSpot虚拟机操作的特定区域，这些区域通常具有特定的系统要求，并且可能需要对系统配置参数的特权访问。其他厂牌的JVM不一定会支持这些选项，并且在未来可能会发生变化。高级选项以-XX开头。

### 3.2 查看支持的参数

**方式一：使用如下命令**

    java -XX:+UnlockExperimentalVMOptions -XX:+UnlockDiagnosticVMOptions -XX:+PrintFlagsInitial   

其中：

- UnlockExperimentalVMOptions：用于解锁实验性参数，如果不加该标记，不会打印实验性参数
- UnlockDiagnosticVMOptions：用于解锁诊断性参数，如果不加该标记，不会打印诊断性参数
- PrintFlagsInitial：打印支持的XX选项，并展示默认值。如需获得程序运行时生效值，用PrintFlagsFinal

**方式二、使用jhsdb**

    jhsdb clhsdb --pid 进程号
    然后输入flags去查看

### 3.3 使用格式

- 如果选项的类型是boolean：那么配置格式是 `-XX:(+/-)选项` 即可。+表示将选项设置为true，-表示设置为false，例如：`-XX:+PrintGC`
- 如果选项的类型不是boolean，那么配置格式是 `-XX:选项=值` ，例如：`-XX:NewRatio=4`

### 3.4 使用示例

    -XX:+PrintGC
    -XX:NewRatio=4

  3.5 JDK  11对高级选项的细分

- [Advanced Runtime Options for Java](https://docs.oracle.com/en/java/javase/11/tools/java.html#GUID-3B1CE181-CD30-4178-9602-230B800D4FAE__BABCBGHF) ：高级运行时选项，控制HotSpot VM的运行时行为
- [Advanced JIT Compiler Options for java](https://docs.oracle.com/en/java/javase/11/tools/java.html#GUID-3B1CE181-CD30-4178-9602-230B800D4FAE__BABDDFII) ：高级JIT编译器选项：控制HotSpot VM执行的JIT编译
- [Advanced Serviceability Options for Java](https://docs.oracle.com/en/java/javase/11/tools/java.html#GUID-3B1CE181-CD30-4178-9602-230B800D4FAE__BABFJDIC) ：高级可服务性选项：系统信息收集与调试支持
- [Advanced Garbage Collection Options for Java](https://docs.oracle.com/en/java/javase/11/tools/java.html#GUID-3B1CE181-CD30-4178-9602-230B800D4FAE__BABFAFAE) ：高级垃圾收集选项：控制HotSpot如何执行垃圾收集

### 3.6 JDK 8对高级选项的细分

- [Advanced Runtime Options](https://docs.oracle.com/javase/8/docs/technotes/tools/windows/java.html#BABCBGHF)
- [Advanced JIT Compiler Options](https://docs.oracle.com/javase/8/docs/technotes/tools/windows/java.html#BABDDFII)
- [Advanced Serviceability Options](https://docs.oracle.com/javase/8/docs/technotes/tools/windows/java.html#BABFJDIC)
- [Advanced Garbage Collection Options](https://docs.oracle.com/javase/8/docs/technotes/tools/windows/java.html#BABFAFAE)

### 3.7 常用高级选项

> **TIPS**
> 
> 完整的高级选项详见：
> 
> **JDK 11**
> 
> - [Advanced Runtime Options for Java](https://docs.oracle.com/en/java/javase/11/tools/java.html#GUID-3B1CE181-CD30-4178-9602-230B800D4FAE__BABCBGHF) ：控制HotSpot VM的运行时行为
> - [Advanced JIT Compiler Options for java](https://docs.oracle.com/en/java/javase/11/tools/java.html#GUID-3B1CE181-CD30-4178-9602-230B800D4FAE__BABDDFII) ：控制HotSpot VM执行的JIT编译
> - [Advanced Serviceability Options for Java](https://docs.oracle.com/en/java/javase/11/tools/java.html#GUID-3B1CE181-CD30-4178-9602-230B800D4FAE__BABFJDIC) ：系统信息收集与调试支持
> - [Advanced Garbage Collection Options for Java](https://docs.oracle.com/en/java/javase/11/tools/java.html#GUID-3B1CE181-CD30-4178-9602-230B800D4FAE__BABFAFAE) ：控制HotSpot如何执行垃圾收集
> 
> 
> **JDK 8**
> 
> - [Advanced Runtime Options](https://docs.oracle.com/javase/8/docs/technotes/tools/windows/java.html#BABCBGHF)
> - [Advanced JIT Compiler Options](https://docs.oracle.com/javase/8/docs/technotes/tools/windows/java.html#BABDDFII)
> - [Advanced Serviceability Options](https://docs.oracle.com/javase/8/docs/technotes/tools/windows/java.html#BABFJDIC)
> - [Advanced Garbage Collection Options](https://docs.oracle.com/javase/8/docs/technotes/tools/windows/java.html#BABFAFAE)

#### 3.7.1 常用高级运行时选项

- 
-XX:ActiveProcessorCount=x

JVM使用多少个CPU核心去计算用于执行垃圾收集或ForkJoinPool线程池的大小

- 
-XX:InitiatingHeapOccupancyPercent=n

老年代大小达到该阈值，就触发Mixed GC，默认值为45

- 
-XX:LargePageSizeInBytes=size

设置用于Java堆的大页面尺寸。单位字节，数值必须是2的次幂。也可在size后追加字母 `k` 或 `K` 表示千字节， `m` 或 `M` 表示兆字节，或 `g` 或 `G` 表示千兆字节

- 
-XX:MaxDirectMemorySize=size

设置`java.nio` 包直接缓冲区分配的最大总大小。以字节为单位也可在size后追加字母 `k` 或 `K` 表示千字节， `m` 或 `M` 表示兆字节，或 `g` 或 `G` 表示千兆字节

- 
-XX:MaxGCPauseMillis=ms

期望的最大停顿时间，默认值200ms

- 
-XX:OnError=string

发生错误时候做某事，string是一个或多个命令，多个命令用;分隔，如果字符串包含空格，则必须将其用引号引起来。例如：

    # 当发生错误时，使用gcore命令创建核心dump文件
    -XX:OnError="gcore %p;dbx - %p"

- 
-XX:OnOutOfMemoryError=string

当发生OutOfMemoryError异常时做某事。配置格式同 `-XX:OnError`

- 
-XX:ParallelGCThreads=n

设置并行阶段的线程数

- 
-XX:+PrintCommandLineFlags

打印命令行标记，默认关闭

- 
-XX:ThreadStackSize=size

设置线程栈的大小，和-Xss等价

- 
-XX:-UseBiasedLocking

禁用偏向锁

- 
-XX:-UseCompressedOops

禁用压缩指针。默认情况下，启用此选项，并且当Java堆大小小于32 GB时，将使用压缩指针。启用此选项后，对象引用将表示为32位偏移量，而非64位指针，这通常会在运行Java堆大小小于32GB的应用程序时提高性能。此选项仅适用于64位JVM。当Java堆大小大于32 GB时，使用 `-XX:ObjectAlignmentInBytes` 选项。

- 
-XX:GCLogFileSize=number

处理大型日志文件，默认为512K

- 
-XX:+UseLargePages

启用大页面内存的使用，默认关闭

- 
-XX:VMOptionsFile=filename

允许用户在文件中指定VM选项。例如 `java -XX:VMOptionsFile=/var/my_vm_options HelloWorld`。

#### 3.7.2 常用高级JIT编译器选项

- 
-XX:+BackgroundCompilation

启用后台编译，默认开启

- 
-XX:CompileCommand=command,method[,option]

在指定方法上执行指定command。command可选项：

- 
break：

在调试JVM时设置一个断点，以便在指定方法编译开始时停止

- 
compileonly

排除所有未指定的所有方法。

- 
dontinline

防止内联指定方法

- 
exclude

排除指定的方法

- 
help

打印 `-XX:CompileCommand` 选项的帮助消息。

- 
inline

尝试内联指定的方法。

- 
log

排除指定方法以外的所有方法的编译日志记录（用 `-XX:+LogCompilation`  打印编译日志）。默认情况下，将对所有编译方法执行日志记录。

- 
option

将JIT编译选项传递给指定的方法，以代替最后一个参数（`option`）。编译选项设置在方法名称之后。例如，要启用StringBuffer类中append()方法的 `BlockLayoutByFrequency` 选项，可使用以下命令：

    -XX:CompileCommand=option,java/lang/StringBuffer.append,BlockLayoutByFrequency

可指定多个编译选项，以逗号或空格分隔。

- 
print

在编译指定的方法后打印生成的汇编代码。

- 
quiet

不打印编译命令。默认情况下，将显示您使用 `-XX:CompileCommand` 选项指定的命令。例如，如果从编译中排除 `String` 类的 `indexOf()` 方法 ，则将以下内容输出到标准输出：

    CompilerOracle: exclude java/lang/String.indexOf

可通过 `-XX:CompileCommand=quiet` 在其他 `-XX:CompileCommand` 选项之前指定该选项来抑制这种情况。

- 
-XX:+DoEscapeAnalysis

开启逃逸分析，默认开启

- 
-XX:+Inline

启用方法内联，默认开启

- 
-XX:InlineSmallCode=size

指定内联的已编译的方法的最大代码大小已字节为单位，默认1000字节

- 
-XX:+LogCompilation

将编译活动记录到当前工作目录的hotspot.log文件中，也可用`-XX:LogFile`选项指定其他日志文件路径和名称，默认关闭。该选项必须与 `-XX:+UnlockDiagnosticVMOptions` 配合使用。

- 
-XX:MaxInlineSize=size

设置要内联的方法的最大字节码大小，以字节为单位

- 
-XX:+PrintAssembly

打印字节码和本地方法的汇编代码，需要hsdis的支持。默认禁用，该选项必须与 `-XX:+UnlockDiagnosticVMOptions` 配合使用。

- 
-XX:+PrintCompilation

编译方法时，将消息打印到控制台来启用JVM的详细诊断输出，默认关闭。

- 
-XX:+PrintInlining

打印哪些方法被内联。默认关闭，该选项必须与 `-XX:+UnlockDiagnosticVMOptions` 配合使用。

- 
-XX:ReservedCodeCacheSize=size

设置JIT编译的代码的最大代码缓存的大小，以字节为单位，默认是240MB，最大不超过2GB，否则会产生错误。另外，除非禁用分层编译，否则XX:-TieredCompilation默认大小为48 MB。该配置不应小于`-XX:InitialCodeCacheSize` 的值。

#### 3.7.3 常用高级可服务性选项

- 
-XX:HeapDumpPath=path

指定堆Dump的文件路径，经常和 `-XX:+HeapDumpOnOutOfMemoryError` 选项配合使用。默认情况下，文HeapDumpPath可显式设置Dump文件的路径，例如：

    -XX:HeapDumpPath=./java_pid%p.hprof

- 
-XX:LogFile=path

指定日志文件的路径，默认情况下文件在当前工作目录中创建，名为hotspot.log 。该选项经常和-XX:+LogCompilation配合使用。

- 
-XX:+UnlockDiagnosticVMOptions

解锁用于诊断JVM的选项，默认关闭

#### 3.7.4 高级垃圾收集选项

略，见垃圾收集器相关选项。

有余力的同学也可以通读下 [Advanced Garbage Collection Options for Java](https://docs.oracle.com/en/java/javase/11/tools/java.html#GUID-3B1CE181-CD30-4178-9602-230B800D4FAE__BABFAFAE)

#### 3.7.5 垃圾收集器相关选项

> **TIPS**
> 
> 这部分选项基于JDK 8整理，理论可适用于JDK 11及更低版本。

各款垃圾收集器相关的JVM参数总结如下表。

| 收集器 | 参数及默认值 | 备注 |
| :- | :- | :- |
| Serial | -XX:+UseSerialGC | 虚拟机在Client模式下的默认值，开启后，使用 Serial + Serial Old 的组合 |
| ParNew | -XX:+UseParNewGC | 开启后，使用ParNew + Serial Old的组合 |
|  | -XX:ParallelGCThreads=n | 设置垃圾收集器在并行阶段使用的垃圾收集线程数，当逻辑处理器数量小于8时，n的值与逻辑处理器数量相同；如果逻辑处理器数量大于8个，则n的值大约为逻辑处理器数量的5/8，大多数情况下是这样，除了较大的SPARC系统，其中n的值约为逻辑处理器的5/16。 |
| Parallel Scavenge | -XX:+UseParallelGC | 虚拟机在Server模式下的默认值，开启后，使用 Parallel Scavenge + Serial Old的组合 |
|  | -XX:MaxGCPauseMillis=n | 收集器尽可能保证单次内存回收停顿的时间不超过这个值，但是并不保证不超过该值 |
|  | -XX:GCTimeRatio=n | 设置吞吐量的大小，取值范围0-100，假设 GCTimeRatio 的值为 n，那么系统将花费不超过 1/(1+n) 的时间用于垃圾收集 |
|  | -XX:+UseAdaptiveSizePolicy | 开启后，无需人工指定新生代的大小（-Xmn）、 Eden和Survisor的比例（-XX:SurvivorRatio）以及晋升老年代对象的年龄（-XX:PretenureSizeThreshold）等参数，收集器会根据当前系统的运行情况自动调整 |
| Serial Old | 无 | Serial Old是Serial的老年代版本，主要用于 Client 模式下的老生代收集，同时也是 CMS 在发生 Concurrent Mode Failure时的后备方案 |
| Parallel Old | -XX:+UseParallelOldGC | 开启后，使用Parallel Scavenge + Parallel Old的组合。Parallel Old是Parallel Scavenge的老年代版本，在注重吞吐量和 CPU 资源敏感的场合，可以优先考虑这个组合 |
| CMS | -XX:+UseConcMarkSweepGC | 开启后，使用ParNew + CMS的组合；Serial Old收集器将作为CMS收集器出现 Concurrent Mode Failure 失败后的后备收集器使用 |
|  | -XX:CMSInitiatingOccupancyFraction=68 | CMS 收集器在老年代空间被使用多少后触发垃圾收集，默认68% |
|  | -XX:+UseCMSCompactAtFullCollection | 在完成垃圾收集后是否要进行一次内存碎片整理，默认开启 |
|  | -XX:CMSFullGCsBeforeCompaction=0 | 在进行若干次Full GC后就进行一次内存碎片整理，默认0 |
|  | -XX:+UseCMSInitiatingOccupancyOnly | 允许使用占用值作为启动CMS收集器的唯一标准，一般和CMSFullGCsBeforeCompaction配合使用。如果开启，那么当CMSFullGCsBeforeCompaction达到阈值就开始GC，如果关闭，那么JVM仅在第一次使用CMSFullGCsBeforeCompaction的值，后续则自动调整，默认关闭。 |
|  | -XX:+CMSParallelRemarkEnabled | 重新标记阶段并行执行，使用此参数可降低标记停顿，默认打开（仅适用于ParNewGC） |
|  | -XX:+CMSScavengeBeforeRemark | 开启或关闭在CMS重新标记阶段之前的清除（YGC）尝试。新生代里一部分对象会作为GC Roots，让CMS在重新标记之前，做一次YGC，而YGC能够回收掉新生代里大多数对象，这样就可以减少GC Roots的开销。因此，打开此开关，可在一定程度上降低CMS重新标记阶段的扫描时间，当然，开启此开关后，YGC也会消耗一些时间。PS. 开启此开关并不保证在标记阶段前一定会进行清除操作，生产环境建议开启，默认关闭。 |
| CMS-Precleaning | -XX:+CMSPrecleaningEnabled | 是否启用并发预清理，默认开启 |
| CMS-AbortablePreclean | -XX:CMSScheduleRemark<br/>EdenSizeThreshold=2M | 如果伊甸园的内存使用超过该值，才可能进入“并发可中止的预清理”这个阶段 |
| CMS-AbortablePreclean | -XX:CMSMaxAbortablePrecleanLoops=0 | “并发可终止的预清理阶段”的循环次数，默认0，表示不做限制 |
| CMS-AbortablePreclean | -XX:+CMSMaxAbortablePrecleanTime=5000 | “并发可终止的预清理”阶段持续的最大时间 |
|  | -XX:+CMSClassUnloadingEnabled | 使用CMS时，是否启用类卸载，默认开启 |
|  | -XX:+ExplicitGCInvokesConcurrent | 显示调用System.gc()会触发Full GC，会有Stop The World，开启此参数后，可让System.gc()触发的垃圾回收变成一次普通的CMS GC。 |
|  | -XX:+UseG1GC | 使用G1收集器 |
|  | -XX:G1HeapRegionSize=n | 设置每个region的大小，该值为2的幂，范围为1MB到32MB，如不指定G1会根据堆的大小自动决定 |
|  | -XX:MaxGCPauseMillis=200 | 设置最大停顿时间，默认值为200毫秒。 |
|  | -XX:G1NewSizePercent=5 | 设置年轻代占整个堆的最小百分比，默认值是5，这是个实验参数。需用-XX:+UnlockExperimentalVMOptions解锁试验参数后，才能使用该参数。 |
|  | -XX:G1MaxNewSizePercent=60 | 设置年轻代占整个堆的最大百分比，默认值是60，这是个实验参数。需用-XX:+UnlockExperimentalVMOptions解锁试验参数后，才能使用该参数。 |
|  | -XX:ParallelGCThreads=n | 设置垃圾收集器在并行阶段使用的垃圾收集线程数，当逻辑处理器数量小于8时，n的值与逻辑处理器数量相同；如果逻辑处理器数量大于8个，则n的值大约为逻辑处理器数量的5/8，大多数情况下是这样，除了较大的SPARC系统，其中n的值约为逻辑处理器的5/16。 |
|  | -XX:ConcGCThreads=n | 设置垃圾收集器并发阶段使用的线程数量，设置n大约为ParallelGCThreads的1/4。 |
|  | -XX:InitiatingHeapOccupancyPercent=45 | 老年代大小达到该阈值，就触发Mixed GC，默认值为45。 |
|  | -XX:G1MixedGCLiveThresholdPercent=85 | Region中的对象，活跃度低于该阈值，才可能被包含在Mixed GC收集周期中，默认值为85，这是个实验参数。需用-XX:+UnlockExperimentalVMOptions解锁试验参数后，才能使用该参数。 |
|  | -XX:G1HeapWastePercent=5 | 设置浪费的堆内存百分比，当可回收百分比小于浪费百分比时，JVM就不会启动Mixed GC，从而避免昂贵的GC开销。此参数相当于用来设置允许垃圾对象占用内存的最大百分比。 |
|  | -XX:G1MixedGCCountTarget=8 | 设置在标记周期完成之后，最多执行多少次Mixed GC，默认值为8。 |
|  | -XX:G1OldCSetRegionThresholdPercent=10 | 设置在一次Mixed GC中被收集的老年代的比例上限，默认值是Java堆的10%，这是个实验参数。需用-XX:+UnlockExperimentalVMOptions解锁试验参数后，才能使用该参数。 |
|  | -XX:G1ReservePercent=10 | 设置预留空闲内存百分比，虚拟机会保证Java堆有这么多空间可用，从而防止对象晋升时无空间可用而失败，默认值为Java堆的10％。 |
|  | -XX:-G1PrintHeapRegions | 输出Region被分配和回收的信息，默认false |
|  | -XX:-G1PrintRegionLivenessInfo | 在清理阶段的并发标记环节，输出堆中的所有Regions的活跃度信息，默认false |
| Shenandoah | -XX:+UseShenandoahGC | 使用UseShenandoahGC，这是个实验参数，需用-XX:+UnlockExperimentalVMOptions解锁试验参数后，才能使用该参数；另外该参数只能在Open JDK中使用，Oracle JDK无法使用 |
| ZGC | -XX:+UseZGC | 使用ZGC，这是个实验参数，需用-XX:+UnlockExperimentalVMOptions解锁试验参数后，才能使用该参数； |
| Epsilon | -XX:+UseEpsilonGC | 使用EpsilonGC，这是个实验参数，需用-XX:+UnlockExperimentalVMOptions解锁试验参数后，才能使用该参数； |


## 4 不应使用的选项(obsolete, deprecated, and removed Options)

这部分选项都尽量不要去使用，这类选项：

JDK 11中可细分为：

- [Deprecated Java Options](https://docs.oracle.com/en/java/javase/11/tools/java.html#GUID-3B1CE181-CD30-4178-9602-230B800D4FAE__DEPRECATEDJAVAOPTIONS-A4E6FB83) ：废弃的选项：使用这些参数时，JDK会接受并执行这些参数，但使用时会有警告
- [Obsolete Java Options](https://docs.oracle.com/en/java/javase/11/tools/java.html#GUID-3B1CE181-CD30-4178-9602-230B800D4FAE__OBSOLETEJAVAOPTIONS-A4E7030A) ：淘汰的选项：使用这些参数时，JDK会接受这些参数，但会直接忽略(相当于加了也没效果)，使用时会有警告
- [Removed Java Options](https://docs.oracle.com/en/java/javase/11/tools/java.html#GUID-3B1CE181-CD30-4178-9602-230B800D4FAE__REMOVEDJAVAOPTIONS-A4E6F213) ：删除的选项：使用这些参数时，会导致错误

JDK 8文档中未做细分。读者可自行前往 [https://chriswhocodes.com/hotspot_options_jdk8.html](https://chriswhocodes.com/hotspot_options_jdk8.html) 搜索选项的状态。

## 参考文档

- [https://docs.oracle.com/en/java/javase/11/tools/java.html](https://docs.oracle.com/en/java/javase/11/tools/java.html)
- [https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html)