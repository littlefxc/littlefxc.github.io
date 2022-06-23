---
title: "JVM 垃圾回收"
date: 2022-03-01T15:15:03+08:00
categories: Java
tags:
- jvm
---

## 概述

- 什么场景下该使用什么垃圾回收策略？
- 垃圾回收发生在哪些区域？
- 对象在什么时候能够被回收？

## 什么场景下该使用什么垃圾回收策略？

- 场景一：在对内存要求苛刻的场景：想办法提高对象的回收效率，多回收掉一些对象，腾出更多内存；
- 场景二：在CPU 使用率高的场景下：降低高并发垃圾回收的频率，让 CPU 更多地去执行你的业务而不是垃圾回收；

## 垃圾回收发生在哪些区域？

JVM 内存结构如下图所示：

![JVM内存结构](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220301152640105.png)

- 虚拟机栈、本地方法栈、程序计数器是线程隔离的，这 3 个区域与线程的生命周期保持一致，同生共死，是不需要考虑垃圾回收的。
- 堆和方法区是线程共享的，才需要考虑。

## 对象在什么时候能够被回收？

就目前来说，有两种算法区判断算法什么时候回收。

### 引用计数法

通过对象的引用计数器来判断对象是否被引用；

如下图所示，对象被引用一次，计数器就加 1 ，但是一旦遇到有循环引用的情况，对象就无法被回收。

![image-20220301153530892](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220301153530892.png)

### 可达性分析

Java并没有采用引用计数法，而是使用了可达性分析。

可达性分析：以根对象（GC Roots）作为起点向下搜索，走过的路径被称为引用链（Reference Chain），如果某个对象到根对象没有引用链相连时，就认为这个对象是不可大的，可以被回收。

如下图所示：

![可达性分析](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220301154408408.png)

#### GC Roots 包括哪些对象

- 虚拟机栈（栈帧中的本地变量表）中引用的对象
- 方法区中类静态属性引用的对象
- 方法区中常量引用的对象
- 本地方法栈中JNI（即 native 方法）引用的对象

#### 那什么是引用？

##### 强引用

- 形如 `Object obj = new Object()` 的引用
- 只要强引用在，永远不会回收被引用的对象，哪怕是内存溢出

##### 软引用

- 形如 `SoftReference<String> str = new SoftReference<String>("hello")`
- 是用来描述一些有用但非必需的对象
- 软引用关联的对象只有在内存不足的时候才会回收

##### 弱引用

- 形如 `WeakReference<String> str = new WeakReference<String>("hello")`
- 弱引用也用来描述非必需对象的
- 被弱引用关联的对象只能生存到下一次垃圾收集发生为止
- 无论当前内存是否足够,都会回收掉只被弱引用关联的对象

##### 虚引用

- 形如
  ```java
  ReferenceQueue<String> queue = new ReferenceQueue<>();
  PhantomReference<String> pr = new PhantomReference<>("hello", queue);
  ```
  
- 个对象是否有虚引用的存在,完全不会对其生存时间构成影响,也无法通过虚引用来取得一个对象实例。

- 为一个对象设置虚引用关联的**唯一目的**只是为了能在这个对象被收集器回收时收到一个系统通知。

### 可达性分析注意点

#### 一个对象即使不可达，也不一定会被回收

即使在可达性分析算法中判定为不可达的对象,也不是“非死不可”的,这时候它们暂时还处于“缓刑”阶段,要真正宣告一个对象死亡,至少要经历两次标记过程：

![image-20220301162604333](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220301162604333.png)

- 如果对象在进行可达性分析后发现没有与GC Roots相连接的引用链,那它将会被第一次标记,随后进行一次筛选,筛选的条件是此对象是否有必要执行finalize()方法。假如对象没有覆盖finalize()方法,或者finalize()方法已经被虚拟机调用过,那么虚拟机将这两种情况都视为“没有必要执行”。

- 如果这个对象被判定为确有必要执行finalize()方法,那么该对象将会被放置在一个名为F-Queue的队列之中,并在稍后由一条由虚拟机自动建立的、低调度优先级的Finalizer线程去执行它们的finalize() 方法。这里所说的“执行”是指虚拟机会触发这个方法开始运行,但并不承诺一定会等待它运行结束。
  这样做的原因是,如果某个对象的finalize()方法执行缓慢,或者更极端地发生了死循环,将很可能导致F-Queue队列中的其他对象永久处于等待,甚至导致整个内存回收子系统的崩溃。finalize()方法是对象逃脱死亡命运的最后一次机会,稍后收集器将对F-Queue中的对象进行第二次小规模的标记,如果对象要在finalize()中成功拯救自己——只要重新与引用链上的任何一个对象建立关联即可,譬如把自己(this关键字)赋值给某个类变量或者对象的成员变量,那在第二次标记时它将被移出“即将回收”的集合;如果对象这时候还没有逃脱,那基本上它就真的要被回收了。

#### 代码演示：

  ```java
  package com.fengxuechao.jvm;
  /**
   * 此代码演示了两点:
   * * 1.对象可以在被GC时自我拯救。
   * * 2.这种自救的机会只有一次,因为一个对象的finalize()方法最多只会被系统自动调用一次
   *
   */
  public class FinalizeEscapeGC {
      public static FinalizeEscapeGC SAVE_HOOK = null;
  
      public static void main(String[] args) throws Throwable {
          SAVE_HOOK = new FinalizeEscapeGC();
          //对象第一次成功拯救自己
          SAVE_HOOK = null;
          System.gc();
          // 因为Finalizer方法优先级很低,暂停0.5秒,以等待它
          Thread.sleep(500);
          if (SAVE_HOOK != null) {
              SAVE_HOOK.isAlive();
          } else {
              System.out.println("no, i am dead :(");
          }
          // 下面这段代码与上面的完全相同,但是这次自救却失败了
          SAVE_HOOK = null;
          System.gc();
          // 因为Finalizer方法优先级很低,暂停0.5秒,以等待它
          Thread.sleep(500);
          if (SAVE_HOOK != null) {
              SAVE_HOOK.isAlive();
          } else {
              System.out.println("no, i am dead :(");
          }
      }
  
      public void isAlive() {
          System.out.println("yes, i am still alive :)");
      }
  
      @Override
      protected void finalize() throws Throwable {
          super.finalize();
          System.out.println("finalize method executed!");
          FinalizeEscapeGC.SAVE_HOOK = this;
      }
  }
  ```

  结果展示：

  ![image-20220301163844090](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220301163844090.png)

事实上假如将代码第24行也就是第二次调用 System.gc() 前没有将SAVE_HOOK 设为 null，那么 SAVE_HOOK 对象将永远不会被回收。

#### finalize() 的建议

- 避免使用finalize()方法，操作不当可能会导致问题；

- finalize() 优先级低，何时会被调用无法确定，因为什么时间发生 GC 不确定；
- 建议使用 try...catch...finally来替代 finalize()。 