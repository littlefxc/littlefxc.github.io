---
title: "JVM 日志"
date: 2022-04-01T15:33:45+08:00
categories: Java
tags:
- jvm
---

JDK8 JVM 垃圾收集日志打印参数

```sh
-Xms50m -Xmx50m -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+PrintGCCause -Xloggc:/Users/fengxuechao/WorkSpace/IdeaProjects/foodie/logs/order_gclog.log
```

JDK8 运行时参数

- `-XX:+TraceClassLoading`：跟踪类加载情况

- `+XX:TraceBiasedLocking`：跟踪偏向锁的情况 

