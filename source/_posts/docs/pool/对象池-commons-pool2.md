---
title: 对象池 commons-pool2
date: 2021-08-01 13:48:21
categories: Java
tags:
- 池化技术
---

# 1 前言

池化技术是性能调优的重要措施，池化的思想是把对象放到池子里面，当要使用的时候从池子里面拿对象，用完之后在放回池子里面。这样可以降低资源分配和释放资源的开销，从而提升性能。

<!-- more -->

在实际的项目中，我们每天都会接触到池化技术，例如：

- 对象池：通过复用对象，减少对象创建、垃圾回收的开销
- 线程池：通过复用线程来提升性能
- 连接池：如数据库连接池、Redis连接池、HTTP连接池，通过复用TCP连接来减少创建和释放连接的时间来提升性能

# 2 对象池 - 适用场景

- 维护一些很大、创建很慢的的对象来提升性能
- 缺点：有学习成本、增加了代码复杂度

# 3 commons-pool2

- Apache 基金会开源的对象池框架
- 官网：https://commons.apache.org/proper/commons-pool/
- Github：https://github.com/apache/commons-pool



commons-pool2 提供了两种对象池：

## 3.1 ObjectPool 接口

ObjectPool 是一个接口

### 3.1.1 实现类介绍

| 实现类                  | 作用                                                         |
| ----------------------- | ------------------------------------------------------------ |
| BaseObjectPool          | 抽象类，用来扩展自己的对象池                                 |
| ErodingObjectPool       | “腐蚀”对象池，代理一个对象池，并基于factor参数，为其添加“腐蚀”行为。归还的对象被腐蚀后，将会丢弃，而不是添加到空闲容量中。 |
| **GenericObjectPool**   | 一个可配置的通用对象池实现                                   |
| ProxiedObjectPool       | 代理一个其它的对象池，并基于动态代理（支持JDK代理和Cglib代理），返回一个代理后的对象。该对象池主要用来增强对池化对象的控制，比如防止在归还该对象后，还继续使用该对象等行为。 |
| SoftReferenceObjectPool | 基于软引用的对象池                                           |
| SynchroizedObjectPool   | 代理一个其它的对象池，并为其提供线程安全的能力               |

### 3.1.2 核心API

| 方法名             | 作用                                                   |
| ------------------ | ------------------------------------------------------ |
| borrowObject()     | 从对象池借对象                                         |
| returnObject()     | 将对象归还到对象池                                     |
| invalidateObject() | 失效一个对象                                           |
| addObject()        | 增加一个空闲对象，该方法适用于使用空闲对象预加载对象池 |
| clear()            | 清空空闲的所有对象，并释放相关资源                     |
| close()            | 关闭对象池，并释放相关资源                             |
| getNumIdIe()       | 获得空闲的对象数量                                     |
| getNumActive()     | 获得被借出对象数量                                     |

## 3.2 KeyedObjectPool

它和ObejectPool的区别在于它是通过 Key 来找对象的。从设计上的角度看，KeyedObjectPool 和 ObjectPool 没有多少区别，它有以下实现类：

| 实现类                     | 作用                      |
| -------------------------- | ------------------------- |
| ErodingKeyedObjectPool     | 类似ErodingObjectPool     |
| GenericKeyedObjectPool     | 类似GenericObjectPool     |
| ProxiedKeyedObjectPool     | 类似ProxiedObjectPool     |
| SynchroizedKeyedObjectPool | 类似SynchroizedObjectPool |

### 3.2.1 PooledObjectFactory

| 实现类                                    | 作用                                                         |
| ----------------------------------------- | ------------------------------------------------------------ |
| BasePooledObjectFactory                   | 抽象类，用于扩展自己的PooledObjectFactory                    |
| PoolUtils.SynchronizedPooledObjectFactory | 内部类，代理一个其它的PooledObjectFactory，实现线程同步，用 `PoolUtils.synchronizedPooledFactory()`创建 |

### 3.2.2 PooledObjectFactory 核心API

| 方案名         | 作用                                                         |
| -------------- | ------------------------------------------------------------ |
| makeObject     | 创建一个对象实例，并将其包装成一个PooledObject               |
| destroyObject  | 销毁对象                                                     |
| validateObject | 校验对象，确保对象池返回的对象是OK的                         |
| activeObject   | 重新初始化对象                                               |
| passivated     | 取消初始化对象。GenriObjectPool 的 addIdIeObject、returnObject、evict调用该方法 |

### 3.2.3 PooledObject

| 实现类                    | 作用                                                         |
| ------------------------- | ------------------------------------------------------------ |
| DefaultPooledObject       | 包装原始对象，实现监控（例如创建时间、使用时间等）、状态跟踪等 |
| PooledSoftReferenceObject | 进一步封装了DefaultPooledObject，用来和SoftReferenceObjectPool配置使用 |

**PooledObject状态**

![image-20210801162749626](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210801162749626.png)

![image-20210801162852280](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210801162852280.png)

### 3.2.2 示例代码 

```java

# 对象
public class Money {
  public static Money init() {
    // 假设对象new非常耗时
    try {
      Thread.sleep(10L);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    return new Money("USD", new BigDecimal("1"));
  }

  private String type;
  private BigDecimal amount;

  public Money(String type, BigDecimal amount) {
    try {
      Thread.sleep(100L);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    this.type = type;
    this.amount = amount;
  }

  // 省略 getter,setter
}

# 对象池工厂类
public class MoneyPooledObjectFactory
  implements PooledObjectFactory<Money> {
  public static final Logger LOGGER =
    LoggerFactory.getLogger(MoneyPooledObjectFactory.class);

  @Override
  public PooledObject<Money> makeObject() throws Exception {
    DefaultPooledObject<Money> object = new DefaultPooledObject<>(
      new Money("USD", new BigDecimal("1"))
    );
    LOGGER.info("makeObject..state = {}", object.getState());
    return object;
  }

  @Override
  public void destroyObject(PooledObject<Money> p) throws Exception {
    LOGGER.info("destroyObject..state = {}", p.getState());
  }

  @Override
  public boolean validateObject(PooledObject<Money> p) {
    LOGGER.info("validateObject..state = {}", p.getState());
    return true;
  }

  @Override
  public void activateObject(PooledObject<Money> p) throws Exception {
    LOGGER.info("activateObject..state = {}", p.getState());
  }

  @Override
  public void passivateObject(PooledObject<Money> p) throws Exception {
    LOGGER.info("passivateObject..state = {}", p.getState());
  }
}

# 测试类
public class CommonsPool2Test {
  public static void main(String[] args) throws Exception {
    GenericObjectPool<Money> pool = new GenericObjectPool<>(new MoneyPooledObjectFactory());
    Money money = pool.borrowObject();
    money.setType("RMB");
    pool.returnObject(money);
  }
}

# 测试结果
  
makeObject..state = IDLE
activateObject..state = ALLOCATED
passivateObject..state = RETURNING

```

## 3.3 commons-pool2 小结

- ObjectPool：对象池
  - 最核心：GenericObjectPool、GenericKeyedObjectPool

- Factory：创建和管理 PooledObject
  - 一般要自己扩展

- PooledObject：包装原有的对象，从而让对象池管理
  - 一般用 DefaultPooledObject

只要掌握这三点，一般就没有问题了

## 3.4 commons-pool2 配置

### 3.4.1 GenericObjectPoolPoolConfig

![image-20210801211018346](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210801211018346.png)

### 3.4.2 AbandonedConfig

![image-20210801211157285](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210801211157285.png)

### 3.4.3 注意点

![image-20210801211531498](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210801211531498.png)

![image-20210801211544503](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210801211544503.png)

## 3.5 Abandon 与 Evict 区别

![image-20210801212906811](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210801212906811.png)

![image-20210801212937024](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210801212937024.png)

![image-20210801213320219](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210801213320219.png)

