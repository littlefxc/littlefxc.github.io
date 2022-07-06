---
title: Unsafe类的使用.md
date: 2022-06-30 16:30:02
categories: Java
tags:
- jvm
---

# Unsafe类的使用

Unsafe可用来直接访问系统内存资源并自主管理，在提升Java运行效率、增强Java语言底层操作能力方面起了很大的作用——可以认为，Unsafe类是Java中留下的后门，提供了一些低层次操作，如直接内存访问、线程调度等。

Unsafe不属于Java标准。 **官方并不建议使用Unsafe，并且从JDK 9开始，官方开始去Unsafe！**
相关Issue：<https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6852936>

因此，Unsafe类对于项目实战，意义并不大。然而目前业界有很多好用的类库大量使用了Unsafe类，例如
`java.util.concurrent.atomic` 包里的一堆类、Netty、Hadoop、Kafka等。所以了解一下还是有好处的。

不同的JDK版本中，Unsafe类也有区别，例如：

  * 在JDK 8中归属于sun.misc包下；
  * 在JDK 11中归属于sun.misc包或jdk.internal.misc下，其中jdk.internal.misc下的Unsafe类功能更强。（应该是从JDK 9开始的，笔者未亲测）

## 快速上手

    import sun.misc.Unsafe;
    import java.lang.reflect.Field;
    
    public class DirectMemoryTest1 {
        private static final int _1MB = 1024 * 1024;
    
        public static void main(String[] args) throws IllegalAccessException {
            //通过反射获取Unsafe类并通过其分配直接内存
            Field unsafeField = Unsafe.class.getDeclaredFields()[0];
            unsafeField.setAccessible(true);
            Unsafe unsafe = (Unsafe) unsafeField.get(null);
            int i = 0;
            while (true) {
                unsafe.allocateMemory(_1MB);
                System.out.println(++i);
            }
        }
    }


## Unsafe类的使用

详见：

  * <https://www.jb51.net/article/140726.htm>
  * <https://blog.csdn.net/ahilll/article/details/81628215>
  * <https://www.jianshu.com/p/dd2be4d3b88e>

## JDK 11如何使用Unsafe类

> **TIPS**
>
>   * 再次强调，实际项目中，除非情非得已，尽量不要用Unsafe类，官方不建议使用
>   * 从JDK 10开始，Unsafe类的部分功能已经被VarHandle取代，建议用VarHandle
>

  * 创建一个JDK 11的项目

  * 在项目源码路径下创建`module-info.java` ，内容如下：
    
    ```java
    module unsafe {
      requires jdk.unsupported;
    }
    ```
    

    创建后，代码结构如下：

    ```java
    |____src
    | |____main
    | | |____java
    | | | |____module-info.java
    | | | |____com
    | | | | |____example
    | | | | | |____demo
    | | | | | | |____UnsafePlayer.java
    | | | | | | |____DirectMemoryTest1.java
    ```

  * 测试代码：
    
    ```java
    import sun.misc.Unsafe;
    import java.lang.reflect.Field;
    
    public class DirectMemoryTest1 {
        private static final int _1MB = 1024 * 1024;
    
        public static void main(String[] args) throws IllegalAccessException {
            //通过反射获取Unsafe类并通过其分配直接内存
            Field unsafeField = Unsafe.class.getDeclaredFields()[0];
            unsafeField.setAccessible(true);
            Unsafe unsafe = (Unsafe) unsafeField.get(null);
            int i = 0;
            while (true) {
                unsafe.allocateMemory(_1MB);
                System.out.println(++i);
            }
      }
    }
    ```

## 参考文档

  * [一篇看懂Java中的Unsafe类](https://www.jb51.net/article/140726.htm)
  * [Using sun.misc.Unsafe in Java 9](https://dzone.com/articles/using-sunmiscunsafe-in-java-9-1)
