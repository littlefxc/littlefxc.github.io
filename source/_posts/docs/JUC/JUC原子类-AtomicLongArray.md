---
title: JUC原子类-AtomicLongArray
date: 2021-01-21 13:33:17
categories: Java
tags: JUC
---

> 转载自https://www.cnblogs.com/skywang12345/p/3514604.html

在"Java多线程系列--“ [Java多线程JUC原子类之 AtomicLong原子类](https://www.notion.so/Java-JUC-AtomicLong-842f5bea464847abbfee4d03a18ca99f) "中介绍过，AtomicLong是作用是对长整形进行原子操作。而AtomicLongArray的作用则是对"长整形数组"进行原子操作。

# AtomicLongArray函数列表

```java
// 创建给定长度的新 AtomicLongArray。
AtomicLongArray(int length)
// 创建与给定数组具有相同长度的新 AtomicLongArray，并从给定数组复制其所有元素。
AtomicLongArray(long[] array)

// 以原子方式将给定值添加到索引 i 的元素。
long addAndGet(int i, long delta)
// 如果当前值 == 预期值，则以原子方式将该值设置为给定的更新值。
boolean compareAndSet(int i, long expect, long update)
// 以原子方式将索引 i 的元素减1。
long decrementAndGet(int i)
// 获取位置 i 的当前值。
long get(int i)
// 以原子方式将给定值与索引 i 的元素相加。
long getAndAdd(int i, long delta)
// 以原子方式将索引 i 的元素减 1。
long getAndDecrement(int i)
// 以原子方式将索引 i 的元素加 1。
long getAndIncrement(int i)
// 以原子方式将位置 i 的元素设置为给定值，并返回旧值。
long getAndSet(int i, long newValue)
// 以原子方式将索引 i 的元素加1。
long incrementAndGet(int i)
// 最终将位置 i 的元素设置为给定值。
void lazySet(int i, long newValue)
// 返回该数组的长度。
int length()
// 将位置 i 的元素设置为给定值。
void set(int i, long newValue)
// 返回数组当前值的字符串表示形式。
String toString()
// 如果当前值 == 预期值，则以原子方式将该值设置为给定的更新值。
boolean weakCompareAndSet(int i, long expect, long update)
```

# 源码分析

AtomicLongArray的代码很简单，下面仅以incrementAndGet()为例，对AtomicLong的原理进行说明。

## incrementAndGet()源码如下：

```java
public final long incrementAndGet(int i) {
    return addAndGet(i, 1);
}
```

说明：incrementAndGet()的作用是以原子方式将long数组的索引 i 的元素加1，并返回加1之后的值。

## addAndGet()源码如下：

```java
public long addAndGet(int i, long delta) {
    // 检查数组是否越界
    long offset = checkedByteOffset(i);
    while (true) {
        // 获取long型数组的索引 offset 的原始值
        long current = getRaw(offset);
        // 修改long型值
        long next = current + delta;
        // 通过CAS更新long型数组的索引 offset的值。
        if (compareAndSetRaw(offset, current, next))
            return next;
    }
}
```

说明：addAndGet()首先检查数组是否越界。如果没有越界的话，则先获取数组索引i的值；然后通过CAS函数更新i的值。

## getRaw()源码如下：

```java
private long getRaw(long offset) {
    return unsafe.getLongVolatile(array, offset);
}
```

说明：unsafe是通过Unsafe.getUnsafe()返回的一个Unsafe对象。通过Unsafe的CAS函数对long型数组的元素进行原子操作。如compareAndSetRaw()就是调用Unsafe的CAS函数，它的源码如下：

## compareAndSetRaw()源码如下：

```java
private boolean compareAndSetRaw(long offset, long expect, long update) {
    return unsafe.compareAndSwapLong(array, offset, expect, update);
}
```

# 用法示例

```java
public class TestAtomicArrayLong {

    public static void main(String[] args) {
        // 新建AtomicLongArray对象
        long[] arrLong = new long[] {10, 20, 30, 40, 50};
        AtomicLongArray ala = new AtomicLongArray(arrLong);

        ala.set(0, 100);
        for (int i=0, len=ala.length(); i<len; i++)
            System.out.printf("get(%d) : %s\\n", i, ala.get(i));

        System.out.printf("%20s : %s\\n", "getAndDecrement(0)", ala.getAndDecrement(0));
        System.out.printf("%20s : %s\\n", "decrementAndGet(1)", ala.decrementAndGet(1));
        System.out.printf("%20s : %s\\n", "getAndIncrement(2)", ala.getAndIncrement(2));
        System.out.printf("%20s : %s\\n", "incrementAndGet(3)", ala.incrementAndGet(3));

        System.out.printf("%20s : %s\\n", "addAndGet(100)", ala.addAndGet(0, 100));
        System.out.printf("%20s : %s\\n", "getAndAdd(100)", ala.getAndAdd(1, 100));

        System.out.printf("%20s : %s\\n", "compareAndSet()", ala.compareAndSet(2, 31, 1000));
        System.out.printf("%20s : %s\\n", "get(2)", ala.get(2));
    }
}
```

结果如下：

```java
get(0) : 100
get(1) : 20
get(2) : 30
get(3) : 40
get(4) : 50
  getAndDecrement(0) : 100
  decrementAndGet(1) : 19
  getAndIncrement(2) : 30
  incrementAndGet(3) : 41
      addAndGet(100) : 199
      getAndAdd(100) : 19
     compareAndSet() : true
              get(2) : 1000

Process finished with exit code 0
```