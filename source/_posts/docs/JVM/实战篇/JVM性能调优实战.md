---
title: JVM性能调优实战
date: 2022-06-23 14:47:17
categories: Java
tags:
- jvm
---

# JVM 性能调优-实战

[TOC]

# 1 CPU过高问题定位

## 1.1 top + jstack

`top`命令可以显示当前系统正在执行的进程的相关信息，包括进程 ID、内存占用率、CPU 占用率等。

### **定位的步骤**

- 使用 top 命令查看当前 CPU 占用率最高的进程，并取得进程ID；

   ![image-20220628164320101](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220628164320101.png)

- 使用 `top -Hp 进程号` 查看进程中的线程的运行信息，获取线程ID;

   ![image-20220628164453884](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220628164453884.png)

- 使用 `printf %x 线程ID` 将线程ID 转换成 16 进制的线程ID；

   ![image-20220628164737066](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220628164737066.png)

- 使用`jstack 进程号 > dump.txt`  dump 一下CPU过高的进程；

- 使用`cat dump.txt |grep -A 30 16进制的线程ID`查看dump日志下中的线程详细信息。

   ![image-20220628165120892](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220628165120892.png)

### **分析**

从dump下来的的日志中可以得到以下信息：

1. 出现问题的线程名是“Thread-0”
2. 有问题的代码是`com.fengxuechao.jvm.inaction.HoldCPUMain$HoldCPUTask.run(HoldCPUMain.java:13)`
3. 找到问题点后，从下文中的问题代码展示中可以发现问题原因在于一个线程占用大量cpu资源用于while循环，而其它3个线程处于空闲状态。

### **问题代码展示**

```java
package com.fengxuechao.jvm.inaction;

/**
 * 该测试简单占用cpu，4个用户线程，一个占用大量cpu资源，3个线程处于空闲状态
 */
public class HoldCPUMain {
    //大量占用cpu
    public static class HoldCPUTask implements Runnable {
        @Override
        public void run() {
            while (true) {
                double a = Math.random() * Math.random();
                System.out.printf("%s %s%n", Thread.currentThread().getId(), a);
            }

        }
    }

    //空闲线程
    public static class LazyTask implements Runnable {
        @Override
        public void run() {
            try {
                while (true) {

                    Thread.sleep(1000);
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        //开启线程，占用cpu
        new Thread(new HoldCPUTask()).start();
        //3个空闲线程
        new Thread(new LazyTask()).start();
        new Thread(new LazyTask()).start();
        new Thread(new LazyTask()).start();
    }
}
```

## 1.2 可能导致CPU占用率过高的场景与解决方案 - 1

### 场景1：无限 while 循环

> 方案1：尽量避免无限循环

> 方案2：让循环执行的慢一点（例如在循环里面加一个 Thread.sleep）

### 场景2：频繁的GC

可以这么想：如果垃圾收集非常频繁的话，说明内存分配的特别快，很快相应的内存区域就满了，于是就导致了下一次的GC。而频繁GC的话，就意味着垃圾收集线程将会频繁的执行，于是，这些垃圾收集线程就会导致CPU的的飙升。

解决方案是**尽量降低GC的频率。**

### 场景3：频繁的创建新对象

> 方案1：合理使用单例

### 场景4：序列化和反序列化

这个场景也非常常见，特别是在一些解析工作的时候特别容易出现。

这篇文章[XStream在反序列化大对象时有严重的性能问题](https://blog.51cto.com/jianshusoft/766400)

序列化和反序列化导致CPU占用率过高，大多数情况都是由于使用类库的方式不合理导致的，因此我们应该：

> 方案1：选择合理的API实现功能。

> 方案2：选择好用的序列化/反序列化类库。

### 场景5：正则表达式

这是因为正则表达式使用了一个叫做 NFA自动机的引擎，这种引擎在字符串匹配的时候会进行回溯（backtracking）。而一旦发生回溯，就可能会导致 CPU 过高的问题了。

比方说这篇文章[一个小小的正则表达式，竟然导致线上CPU 100%异常！](https://mp.weixin.qq.com/s/4fppT7vogJ_H0nxfi8JdCQ)就描述了作者在用正则表达式的时候，出现了一个CPU百分之一百的问题，他的解决方案是改写正则表达式，降低回溯的发生，这样就可以减少CPU的占用率。感兴趣的同学可以去看看这篇文章。

> 方案1：减少字符串匹配期间执行的回溯。

### 场景6：频繁的线程上下文切换

如果线程的上下文频繁切换的话，也可能导致CPU占用率过高。

比方说，现在我们的应用有很多的线程，这些线程在做争抢，线程的状态老是在 Running 和 Block 之间去切换，而一旦这个切换特别频繁的话，就可能会导致导致CPU占用率过高。

> 方案1：降低切换的频率
>
> 这个说起来比较简单，但事实上，需要结合你的业务做一些代码改造，而改造的难度取决于业务的复杂度，所以不要小看降低切换的频率这几个字。

# 2 内存溢出

内存溢出是实际项目中经常会遇到的问题，因此本节内容专门探讨内存溢出如何定位？如何解决？

内存溢出可以做个细分，主要包括：

- 堆内存溢出
- 栈内存溢出
  - 虚拟机栈溢出
  - 本地方法栈溢出
- 方法区溢出
- 直接内存溢出

## 2.1 堆内存溢出

考虑到定位堆内存溢出的问题，如果结合业务去讲解的话，不是那么容易复现，所以我们不如运行一段简单的示例代码让他产生堆内存溢出的的问题，然后用两种办法一步一步快速定位造成堆内存溢出的原因，之后我们总结常见堆内存溢出的场景与解决方案。

### 问题代码演示

```java
package com.fengxuechao.jvm.inaction;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

/**
 * -Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
 */
public class HeapOOMTest {
    private List<String> oomList = new ArrayList<>();

    public static void main(String[] args) {
        HeapOOMTest oomTest = new HeapOOMTest();
        while (true) {
            oomTest.oomList.add(UUID.randomUUID().toString());
        }
    }
}
```

这段代码非常简单，就是不断往列表中不断添加元素。

为什么要加 `-XX:+HeapDumpOnOutOfMemoryError`？是因为实际在生产环境中，可能进程已经挂了。

![image-20220629142924723](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220629142924723.png)

![image-20220629143010112](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220629143010112.png)



### 场景1：内存泄漏

这种情况下，应该借助工具（比如说：MAT、VisualVM、JProfile）去查看泄漏对象到对象的引用列，向上图演示的那样，找到这个泄漏对象是通过怎样的路径跟哪个对象关联，从而导致这个对象没有被回收掉，只要能够定位到泄漏对象的创建位置，那么就能迅速找到导致内存泄漏的代码了。

### 场景2：非内存泄漏

你的项目并不存在内存泄漏的问题，也就是说经过分析我们发现目前内存里的对象都是必须存活的。

那么这个时候，应该根据机器的配置考虑去调大 `-Xmx`、`-Xms`，为应用分配更多的内存。

再一般的，你也可以检查下你的代码，看是不是有些代码的生命周期太长，又或者存储结构不合理，有的时候可能换一个存储结构去存储对象，能够节省很多空间。

示例：

```java
GET /items/comments?itemId=1&page=1&pageSize=100000

可以在实际项目中对 pageSize 做一个限制，例如：
  
if (pageSize > 100) {
  pageSize = 100;
} 
```

## 2.2 栈内存溢出

关于虚拟机栈、本地方法栈在Java虚拟机规范中是这样描述的：

- Java虚拟机规范
  - 如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出 StackOverflowError
  - 如果虚拟机的栈内存允许动态扩展，当无法申请到足够内存时，将抛出 OutOfMemoryError

虚拟机规范是这样定义的，但是对于我们经常打交道的 **Hotspot 虚拟机**来说，有两点意外：

- 栈内存不可扩展
- 不区分虚拟机栈、本地方法栈，统一用 Xss 设置栈的大小
  - 有的虚拟机可以用 Xss 设置虚拟机栈，Xoss 设置本地方法栈

下面进入实战，来一段代码：

### 代码1:

```java
package com.fengxuechao.jvm.inaction;

// 默认配置：18469
// Xss200k：1239
public class StackOOMTest1 {
    private int stackLength = 1;

    private void stackLeak() {
        stackLength++;
        this.stackLeak();
    }

    public static void main(String[] args) {
        StackOOMTest1 oom = new StackOOMTest1();
        try {
            oom.stackLeak();
        } catch (Throwable e) {
            System.out.println("stack length:" + oom.stackLength);
            throw e;
        }
    }
}
```

这段代码主要是关于递归调用的问题，一旦抛异常，就打印栈的深度。

当然在不同的环境下，栈的深度也是不一样的。

之所以会有不同的结果，很好理解，每调用一次方法就会创建一个栈帧，而栈的大小越小，整个栈里面能够容纳的栈帧个数也会越少，所以能够递归的次数也就越少。

![image-20220629171740002](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220629171740002.png)

### 代码2:

```java
package com.fengxuechao.jvm.inaction;

// 默认配置：565
// Xss200k：65
public class StackOOMTest2 {
    private int stackLength = 1;

    private void stackLeak() {
        long unused1, unused2, unused3, unused4, unused5,
            unused6, unused7, unused8, unused9, unused10,
            unused11, unused12, unused13, unused14, unused15,
            unused16, unused17, unused18, unused19, unused20,
            unused21, unused22, unused23, unused24, unused25,
            unused26, unused27, unused28, unused29, unused30,
            unused31, unused32, unused33, unused34, unused35,
            unused36, unused37, unused38, unused39, unused40,
            unused41, unused42, unused43, unused44, unused45,
            unused46, unused47, unused48, unused49, unused50,
            unused51, unused52, unused53, unused54, unused55,
            unused56, unused57, unused58, unused59, unused60,
            unused61, unused62, unused63, unused64, unused65,
            unused66, unused67, unused68, unused69, unused70,
            unused71, unused72, unused73, unused74, unused75,
            unused76, unused77, unused78, unused79, unused80,
            unused81, unused82, unused83, unused84, unused85,
            unused86, unused87, unused88, unused89, unused90,
            unused91, unused92, unused93, unused94, unused95,
            unused96, unused97, unused98, unused99, unused100 = 0;
        stackLength++;
        this.stackLeak();
    }

    public static void main(String[] args) {
        StackOOMTest2 oom = new StackOOMTest2();
        try {
            oom.stackLeak();
        } catch (Error e) {
            System.out.println("stack length:" + oom.stackLength);
            throw e;
        }
    }
}
```

代码 2 和代码 1 几乎一样，只不过在`stackLeak()`方法里面增加了一百个变量。

> 为什么代码2 比代码1 的递归次数要少很多呢？
>
> 这是因为局部变量是存放在栈帧里的局部变量表的，代码1 中没有局部变量，代码2 中有大量局部变量，因此代码2 中的每个栈帧占用的内存比代码1 中的都要大一些，所以对于相同的栈配置，能够容纳的栈帧就会少一些，因此代码2 的递归次数就会少一些。

### 代码3:

```java
package com.fengxuechao.jvm.inaction;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.atomic.AtomicInteger;

/**
 * 注：
 * 本示例可能会造成机器假死，请慎重测试！！！
 * 本示例可能会造成机器假死，请慎重测试！！！
 * 本示例可能会造成机器假死，请慎重测试！！！
 * =======
 * -Xms=10m -Xmx=10m -Xss=144k -XX:MetaspaceSize=10m
 * =======
 * cat /proc/sys/kernel/threads-max
 * - 作用：系统支持的最大线程数，表示物理内存决定的理论系统进程数上限，一般会很大
 * - 修改：sysctl -w kernel.threads-max=7726
 * =======
 * cat /proc/sys/kernel/pid_max
 * - 作用：查看系统限制某用户下最多可以运行多少进程或线程
 * - 修改：sysctl -w kernel.pid_max=65535
 * =======
 * cat /proc/sys/vm/max_map_count
 * - 作用：限制一个进程可以拥有的VMA(虚拟内存区域)的数量，虚拟内存区域是一个连续的虚拟地址空间区域。
 * 在进程的生命周期中，每当程序尝试在内存中映射文件，链接到共享内存段，或者分配堆空间的时候，这些区域将被创建。
 * - 修改：sysctl -w vm.max_map_count=262144
 * =======
 * * ulimit –u
 * * - 作用：查看用户最多可启动的进程数目
 * * - 修改：ulimit -u 65535
 *
 */
public class StackOOMTest3 {
    public static final Logger LOGGER = LoggerFactory.getLogger(StackOOMTest3.class);
    private AtomicInteger integer = new AtomicInteger();

    private void dontStop() {
        while (true) {
        }
    }

    public void newThread() {
        while (true) {
            Thread thread = new Thread(new Runnable() {
                @Override
                public void run() {
                    dontStop();
                }
            });
            thread.start();
            LOGGER.info("线程创建成功，threadName = {}", thread.getName());

            integer.incrementAndGet();
        }
    }

    public static void main(String[] args) {
        StackOOMTest3 oom = new StackOOMTest3();
        try {
            oom.newThread();
        } catch (Throwable throwable) {
            LOGGER.info("创建的线程数：{}", oom.integer);
            LOGGER.error("异常发生", throwable);
            System.exit(1);
        }
    }
}
```

### 如何运行更多线程？

- 减少Xss配置 

- 栈能分配的内存

  - 机器总内存 - 操作系统内存 - 堆内存 - 方法区内存 - 程序计数器内存 - 直接内存

    我们应该想办法尽量减少这些内存的占用，这样栈能分配的内存就会越大

- 尽量杀死其他程序

- 操作系统对线程数目的限制

# 3 方法区溢出

先简单回顾下方法区的知识点：

- 方法区和Java堆一样，是线程共享的，用来存储被虚拟机加载的类型信息、常量、静态变量等

![image-20220630100122633](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220630100122633.png)

Java8 之前方法区是有持久代去实现的，从JDK8之后，持久代被元空间给代替了，同时方法区也成为了一个逻辑上的概念。

## 3.1 代码演示

```java
package com.fengxuechao.jvm.inaction;

import org.springframework.cglib.proxy.Enhancer;
import org.springframework.cglib.proxy.MethodInterceptor;
import org.springframework.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

/**
 * -XX:MetaspaceSize=10m -XX:MaxMetaspace=10m
 */
public class MethodAreaOOMTest2 {
    /**
     * CGLib：https://blog.csdn.net/yaomingyang/article/details/82762697
     *
     * @param args 参数
     */
    public static void main(String[] args) {
        while (true) {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(Hello.class);
            enhancer.setUseCache(false);
            enhancer.setCallback(new MethodInterceptor() {
                public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
                    System.out.println("Enhanced hello");
                    // 调用Hello.say()
                    return proxy.invokeSuper(obj, args);
                }
            });
            Hello enhancedOOMObject = (Hello) enhancer.create();
            enhancedOOMObject.say();
            System.out.println(enhancedOOMObject.getClass().getName());
        }
    }
}

class Hello {
    public void say() {
        System.out.println("Hello Student");
    }
}
```

这段代码主要是用 Cglib 对 Hello 类的 say 方法进行增强。

![image-20220630102738369](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220630102738369.png)

增加对元空间的限制`-XX:MetaspaceSize=10m -XX:MaxMetaspace=10m`后，会报一个元空间溢出的异常。

![image-20220630100507596](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220630100507596.png)

方法区一部分在堆内存，另外一部分在元空间。

## 3.2 方法区总结1

- 不同的JDK版本，方法区存放结构不同，相同的代码报错也可能不同

## 3.3 方法区总结2 - 方法区溢出的场景

- 常量池里对象太大
- 加载的类的“种类”太多
  - 动态代理的操作库生成了大量的动态类
  - JSP项目：因为JSP是在第一次被访问的时候才会被编译成Java类
  - 脚本语言动态类加载。感兴趣的可以看看这篇文章[Metaspace泄漏排查](https://developer.aliyun.com/article/603830)

## 3.4 如何避免方法区溢出

- 根据JDK版本，为常量池（主要是字符串常量池）保留足够空间

  - JDK6：设大 PermSize、MaxPermSize

  - \>=JDK7：设大 Xms、Xmx

- 防止类加载过多导致的溢出

  - <=JDK7：设大 PermSize、MaxPermSize

  - \>= JDK8：留空元空间相关的配置，或设置合理大小的元空间

    元空间是用本地内存去实现的，只要机器的本地内存足够，一般是不会出现问题的。


元空间相关的配置，请看下表

![image-20220630143225611](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220630143225611.png)

| 属性                      | 作用                                                         | 默认值             |
| :------------------------ | ------------------------------------------------------------ | ------------------ |
| -XX:MetaspaceSize         | 元空间的初始值，元空间占用达到该值就会触发垃圾收集，进行类型卸载，同时，收集器会自动调整该值。如果能够释放空间，就会自动降低该值；如果释放空间很少，那么在不超过XX:MaxMetaspaceSize的情况下，可适当提高该值。 | 21810376字节       |
| -XX:MaxMetaspaceSize      | 元空间最大值                                                 | 受限于本地内存大小 |
| -XX:MinMetaspaceFreeRatio | 垃圾收集后，计算当前元空间的空闲百分比，如果小于该值，就增加元空间的大小 | 40%                |
| -XX:MaxMetaspaceFreeRatio | 垃圾收集后，计算当前元空间的空闲百分比，如果大于该值，就减少元空间的大小 | 70%                |
| -XX:MinMetaspaceExpansion | 元空间增长时的最小幅度                                       | 3407847字节        |
| -XX:MaxMetaspaceExpansion | 元空间增长时的最大幅度                                       | 5452592字节        |

# 4 直接内存溢出

![image-20220630153503566](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220630153503566.png)

直接内存

- 《Java 虚拟机规范》并没有定义这块区域
- 不属于虚拟机运行时数据区

## 什么是直接内存？

- 直接内存是一块直接有操作系统直接管理的内存，也叫堆外内存；

- 可以使用 Unsafe 或 ByteBuffer 分配直接内存；
- 可用 -XX:MaxDirectMemorySize 控制，默认是0，表示不限制。

## 为什么要用直接内存？

- 性能优势

  直接内存里的对象读写起来要比堆内存里的要快很多

- [Java直接内存与非直接内存性能测试](https://www.cnblogs.com/xing901022/p/5243657.html?spm=a2c6h.12873639.0.0.3045438ftsvnlq)

## 直接内存使用场景

- 有很大的数据需要存储，生命周期很长
- 有频繁的IO操作，比如并发网络通信

## 实战：分配直接内存

感兴趣的同学可以去看这篇文章[Unsafe类的使用](https://blog.csdn.net/Little_fxc/article/details/125543369?spm=1001.2014.3001.5502)

### 示例1 - Unsafe

```java
package com.fengxuechao.jvm.inaction;
import sun.misc.Unsafe;
import java.lang.reflect.Field;

public class DirectMemoryTest1 {
    private static final int MB_1 = 1024 * 1024;

    public static void main(String[] args) throws IllegalAccessException, NoSuchFieldException {
        //通过反射获取Unsafe类并通过其分配直接内存
        Field unsafeField = Unsafe.class.getDeclaredFields()[0];
        unsafeField.setAccessible(true);
        Unsafe unsafe = (Unsafe) unsafeField.get(null);

        // 分配1M内存，并返回这块内存的起始地址
        long address = unsafe.allocateMemory(MB_1);

        // 向内存地址中设置对象
        unsafe.putByte(address, (byte) 1);

        // 从内存中获取对象
        byte aByte = unsafe.getByte(address);
        System.out.println(aByte);

        // 释放内存
        unsafe.freeMemory(address);
    }
}
```

### 示例2 - ByteBuffer

```java
package com.fengxuechao.jvm.inaction;

import java.nio.ByteBuffer;

public class DirectMemoryTest2 {
    private static final int ONE_MB = 1024 * 1024;

    /**
     * ByteBuffer参考文档：
     * https://blog.csdn.net/z69183787/article/details/77102198/
     *
     * @param args args
     */
    public static void main(String[] args) {
        ByteBuffer buffer = ByteBuffer.allocateDirect(ONE_MB);
        // 相对写，向position的位置写入一个byte，并将postion+1，为下次读写作准备
        buffer.put("abcde".getBytes());
        buffer.put("fghij".getBytes());

        // 转换为读取模式
        buffer.flip();

        // 相对读，从position位置读取一个byte，并将position+1，为下次读写作准备
        // 读取第1个字节(a)
        System.out.println((char) buffer.get());

        // 读取第2个字节
        System.out.println((char) buffer.get());

        // 绝对读，读取byteBuffer底层的bytes中下标为index的byte，不改变position
        // 读取第3个字节
        System.out.println((char) buffer.get(2));
    }
}
```

### 小结

要想分配直接内存，可以使用：

- Unsafe.allocateMemory(size)
- ByteBuffer.allocateDirect(size)

## 直接内存溢出的问题

### Unsafe 演示代码

```java
package com.fengxuechao.jvm.inaction;
import sun.misc.Unsafe;
import java.lang.reflect.Field;

// 1. Unsafe导致直接内存溢出报错没有小尾巴
// 2. -XX:MaxDirectMemorySize=100m对Unsafe不起作用
public class DirectMemoryTest3 {
    private static final int GB_1 = 1024 * 1024 * 1024;

    public static void main(String[] args) throws IllegalAccessException, NoSuchFieldException {
        //通过反射获取Unsafe类并通过其分配直接内存
        Field unsafeField = Unsafe.class.getDeclaredFields()[0];
        unsafeField.setAccessible(true);
        Unsafe unsafe = (Unsafe) unsafeField.get(null);

        int i = 0;
        while (true) {
            unsafe.allocateMemory(GB_1);
            System.out.println(++i);
        }
    }
}
```

### ByteBuffer 演示代码

```java
package com.fengxuechao.jvm.inaction;
import java.nio.ByteBuffer;

// 1. ByteBuffer直接内存溢出报错是java.lang.OutOfMemoryError: Direct buffer memory
// 2. -XX:MaxDirectMemorySize对ByteBuffer有效
public class DirectMemoryTest4 {
    private static final int GB_1 = 1024 * 1024 * 1024;

    /**
     * ByteBuffer参考文档：
     * https://blog.csdn.net/z69183787/article/details/77102198/
     *
     * @param args args
     */
    public static void main(String[] args) {
        int i= 0;
        while (true) {
            ByteBuffer buffer = ByteBuffer.allocateDirect(GB_1);
            System.out.println(++i);
        }
    }
}
```

## 直接内存总结

- 堆Dump文件看不出问题或者比较小，可考虑直接内存溢出的问题
- 配置内存时，应该给直接内存预留足够的空间
- 使用Unsafe分配直接内存时
  - 溢出时报：java.lang.OutOfMemory
  - -XX:MaxDirectMemorySize=100m对Unsafe不起作用
- 使用ByteBuffer分配直接内存时
  - 溢出时报：java.lang.OutOfMemoryError: Direct buffer memory
  - -XX:MaxDirectMemorySize 起作用
  - ByteBuffer底层也是用Unsafe去分配的

#  5 代码缓存区（Code Cache）满

- 用来存储编译后的代码

控制代码缓存区的参数，如下表所示：

| 属性                             | 作用                                                         | 默认值                                     |
| -------------------------------- | ------------------------------------------------------------ | ------------------------------------------ |
| -XX: InitialCodeCacheSize        | 设置代码缓存区的初始大小，以`java -XX:+PrintFlagsFinal |grep Initialcodecachesize` 结果为准 | 不同操作系统、不同编译器的值不同           |
| -XX:ReservedCodeCacheSize        | 设置代码缓存区的最大大小，以`java xx:+PrintFlagsFinal| grep Reservedcodecachesize `结果为准 | 不同版本不同，JDK864位、JDK 1164位都是240M |
| -XX:-PrintCodeCache              | 在JVM停止时打印代码缓存的使用情况                            | 关闭                                       |
| -XX:-PrintCodeCacheOnCompilation | 每当方法被编译后，就打印一下代码缓存区的使用情况             | 关闭                                       |
| -XX:+UseCodeCacheFlushing        | 代码缓存区即将耗尽时，尝试回收一些早期编译、很久末被调用的方法 | 打开                                       |
| -XX:-SegmentedCodeCache          | 是否使用分段的代码缓存区，默认关闭，表示使用整体的代码缓存区 | 关闭                                       |

## 代码缓存区总结

- 设置合理的代码缓存区大小，可以根据 jconsole 中的 "Code Cache" 来设置合理的值

  ![image-20220630172838035](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220630172838035.png)

- 如果项目平时性能很OK，但突然出现性能下降，业务没有问题，可排查是否由代码缓存区满所导致
