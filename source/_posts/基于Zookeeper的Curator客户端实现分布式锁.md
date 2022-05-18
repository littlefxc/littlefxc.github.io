---
title: 基于Zookeeper的Curator客户端实现分布式锁
date: 2021-05-16 11:54:06
tags:
- zookeeper
- 分布式锁
---

# 前言

curator 是Java语言实现的增强版zookeeper客户端详情见[官网](http://curator.apache.org)

curator 的使用步骤：

1. 引入curator客户端
2. curator 已经实现了分布式锁的方法
3. 直接调用即可

<!-- more -->

# 单元测试

```java
@Test
public void testCuratorLock(){
    RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
    CuratorFramework client = CuratorFrameworkFactory.newClient("localhost:2181", retryPolicy);
    client.start();
    InterProcessMutex lock = new InterProcessMutex(client, "/order");
    try {
        if ( lock.acquire(30, TimeUnit.SECONDS) ) {
            try  {
                log.info("我获得了锁！！！");
            }
            finally  {
                lock.release();
            }
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
    client.close();
}
```

