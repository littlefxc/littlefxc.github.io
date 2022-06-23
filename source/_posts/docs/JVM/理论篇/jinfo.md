---
title: 内置故障排查工具-jinfo
date: 2021-08-10 11:20:40
categories: Java
tags:
- jvm
---

# jinfo

## 作用

jinfo全称Java Configuration Info，主要用来查看与调整JVM参数。

> **TIPS**
>
> - 此命令是实验性的，不受支持，对于JDK 9及更高版本，部分功能可使用 `jhsdb jinfo` 代替，也可用jcmd代替。
> - 部分JDK版本的jinfo命令对Windows支持比较有限，参数较少。本文为了更加接近生产环境，都是基于类Unix操作系统编写的。如果在Windows操作系统下测试，应以jinfo -h的结果为准。

## 使用说明

命令如下：

```
➜ jinfo -h
Usage:
    jinfo <option> <pid>
       (to connect to a running process)

where <option> is one of:
    -flag <name>         打印指定参数的值
    -flag [+|-]<name>    启用/关闭指定参数
    -flag <name>=<value> 将指定的参数设置为指定的值
    -flags               打印VM参数
    -sysprops            打印系统属性（笔者注：系统属性打印的是System.getProperties()的结果）
    <no option>          打印VM参数及系统属性
    -? | -h | --help | -help to print this help message
12345678910111213
```

## 使用示例

### 查看参数

示例1：打印42342这个进程的VM参数及Java系统属性：

```
jinfo 42342
```

示例2：打印42342这个进程的Java系统属性

```
jinfo -sysprops 42342
```

示例3：打印42342这个进程的VM参数

```
jinfo -flags 42342
```

示例4：打印42342这个进程ConcGCThreads参数的值

```
jinfo -flag ConcGCThreads 42342
```

> **拓展知识**
>
> 要想查看JVM参数，也可在启动时，指定 `-XX:+PrintFlagsFinal` ，这样会在启动时将JVM参数打印到日志。

### 动态修改参数

示例5：将42342这个进程的PrintClassHistogram设置为false

```
jinfo -flag -PrintClassHistogram 42342
```

示例6：将42342这个进程的MaxHeapFreeRatio设置为80

```
jinfo -flag MaxHeapFreeRatio=80 42342
```

> **TIPS**
>
> 虽然可用jinfo动态修改VM参数，但并非所有参数都支持动态修改，如果操作了不支持的修改的参数，将会报类似如下的异常：
>
> ```
> Exception in thread "main" com.sun.tools.attach.AttachOperationFailedException: flag 'xxx' cannot be changed
> ```
>
> 使用如下命令显示出来的参数，基本上都是支持动态修改的：
>
> ```
> java -XX:+PrintFlagsInitial | grep manageable
> ```
