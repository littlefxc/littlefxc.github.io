---
title: JVM-内存结构
date: 202-02-23 20:49:45
categories: Java
tags:
- jvm
---


## 1.1 内存结构

![https://gitee.com/littlefxc/oss/raw/master/images/JVM内存结构.png](https://gitee.com/littlefxc/oss/raw/master/images/JVM内存结构.png)

- 线程共享：堆，方法区
- 线程隔离：虚拟机栈，本地方法栈，程序计数器

<!-- more -->

### 1.1.1 堆

堆又做了细分如下图所示：

![https://gitee.com/littlefxc/oss/raw/master/images/JVM内存结构-堆.png](https://gitee.com/littlefxc/oss/raw/master/images/JVM内存结构-堆.png)

JDK8 之前堆分为新生代、老年代和持久代（也叫永久代），其中新生代中又有伊甸园和存活区，而存活区又分为 “From survivor” 和 “To survivor”。

JDK8 之后，持久代被废弃，由元空间代替，而元空间并不是堆内存的一部分，元空间是本地内存。

### 1.1.2 虚拟机栈

虚拟机栈是线程独享的，当创建一个现成的时候就会创建虚拟机栈。

![https://gitee.com/littlefxc/oss/raw/master/images/JVM内存结构-虚拟机栈.png](https://gitee.com/littlefxc/oss/raw/master/images/JVM内存结构-虚拟机栈.png)

- 虚拟机栈由栈帧组成。
- 每一次方法调用都会创建一个栈帧，然后去压栈。
- 方法返回的时候代表栈帧的出栈操作。
- 栈帧里面包含一系列数据：局部变量表、操作数栈、指向运行时常量池的引用、方法返回地址和动态链接

### 1.1.2 本地方法栈

虚拟机栈中放的是Java 方法，而本地方法栈放的是 native 方法（如 UnSafe类）。

### 1.1.4 程序计数器

程序计数器用来记录各个字节码执行的字节码的地址，像分支、循环、跳转、异常、线程恢复等等操作都需依赖程序计数器。

- **为什么需要程序计数器?**

  这是因为Java 是个多线程语言，当执行的线程数量超过CPU核心的时候，线程之间就会根据时间片争抢CPU资源。例如，某个线程它的任务还没有执行完成，CPU 就被其它线程抢走，如果之后又轮到这个线程执行任务，那么就得知道从哪里继续执行任务，所以会为每一个线程分配一个程序计数器，用来记录它下一条指令是什么等等。

### 1.1.5 方法区

![https://gitee.com/littlefxc/oss/raw/master/images/JVM内存结构-方法区.png](https://gitee.com/littlefxc/oss/raw/master/images/JVM内存结构-方法区.png)

方法区主要包括 4 个部分：类信息、运行时常量池、字符串常量池和静态变量。

方法区主要存放的是虚拟机加载的类相关的信息。

从上图中可以看到好几种常量池：静态常量池、运行时常量池和字符串常量池。下面来分析一下这 3 种常量池的作用：

- 静态常量池

   也叫 class 文件常量池，主要用来存放：

   - 字面量：例如，文本字符串、final 修饰的常量
   - 符号引用：例如，类和接口的全限定名、字段的名称和描述符、方法的名称和描述符

- 运行时常量池

   当类加载到内存中后，JVM 就会将静态常量池中的内容存放到运行时的常量池中。运行时常量池里面存储的主要是编译期间生成的字面量、符号引用等等。

- 字符串常量池

   也可以理解称运行时常量池分出来的一部分，类加载到内存的时候，字符串会存到字符串常量池里面

**为什么要用元空间代替持久代？**

- 一方面是 Orcale 把 Hotspot 虚拟机和 JRockit 虚拟机收购了，而 JRockit 虚拟机压根就没有永久代的概念。于是为了融合Hotspot 虚拟机和 JRockit 虚拟机，干脆就把它去掉了

- 另一方面是持久代在使用过程中还是很容易发生故障的

  相信很多人都遇到过这种异常 `java.lang.OutOfMemoryError: PermGen`。在之前的版本中，字符串常量池存在于永久代中，在大量使用字符串的情况下，非常容易出现OOM的异常。此外，J**VM加载的class的总数，方法的大小**等都很难确定，因此对永久代大小的指定难以确定。太小的永久代容易导致永久代内存溢出，太大的永久代则容易导致虚拟机内存紧张。

- 元空间(Metaspace)，不再与堆连续，而是直接存在于本地内存中，也就是机器的内存。理论上机器内存有多大，元空间的野心就有多大。

当然，还有很多更多深层次的原因，可以参考这篇博文[Metaspace 之一：Metaspace整体介绍（永久代被替换原因、元空间特点、元空间内存查看分析方法）](https://www.cnblogs.com/duanxz/p/3520829.html)

**示例：**

```java
public class JVMTest1 {
  public static void main(String[] args) {
    Demo demo = new Demo("aaa");
    demo.printName();
  }
}

class Demo {
  private String name;
  public Demo(String name) {
    this.name = name;
  }
  
  public void printName() {
    System.out.println(this.name);
  }
}
```

上面这段代码的内存分布大致是这样的：

![image-20210223220317252](https://gitee.com/littlefxc/oss/raw/master/images/image-20210223220317252.png)

1. 在启动的时候首先将类加载到方法区，要加载两个类分别是 JVMTest1.class 和 Demo.class 。

2. 当创建 Demo 对象的时候，首先会创建一个局部变量 demo 放在栈里面并指向到一个引用，而真正的 Demo 对象会存储到堆里面。

3. 最后执行 printName 方法。