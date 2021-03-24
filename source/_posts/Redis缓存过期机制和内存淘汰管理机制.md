---
title: Redis缓存过期机制和内存淘汰管理机制
date: 2021-01-11 11:51:36
categories: 缓存
tags:
- redis
---

转载自 https://blog.csdn.net/qq_36986015/article/details/106802555

# **1. 缓存过期机制**

Redis可以通过设置一个过期时间expire来处理缓存，其中处理方式有两种：

1. （主动）定期删除，Redis会抽查随机的key，默认1秒十次，一旦抽查的key过期了，就会给删除，配置的属性在redis.conf中，hz等于10，表示1秒抽查10次

    ```
    hz 10
    ```

2. （被动）惰性删除，key到期后不去主动检测，而是请求访问到这个key之后，会检查下是否过期，这样就不会太消耗CPU资源，缺点是一直占用着内存

# **2. 内存淘汰管理机制**

因为计算机的内存是有限的，在部署Redis的同时，也可能部署其他的中间件如RabbitMQ、Kafka等等，为了给其他中间件预留内存空间，Redis服务启动可以设置一个最大内存`maxmemory`，到达阈值后，Redis会清理在内存里永久存在的没有过期时间的key，处理机制如下：

```
maxmemory ：当内存已使用率到达，则开始清理缓存
maxmemory-policy : 清理缓存的策略
```

清理缓存的策略如下所示：

- noeviction：旧缓存永不过期，新缓存设置不了，返回错误
- allkeys-lru：清除最少用的旧缓存，然后保存新的缓存（推荐使用）
- allkeys-random：在所有的缓存中随机删除（不推荐）
- volatile-lru：在那些设置了expire过期时间的缓存中，清除最少用的旧缓存，然后保存新的缓存
- volatile-random：在那些设置了expire过期时间的缓存中，随机删除缓存
- volatile-ttl：在那些设置了expire过期时间的缓存中，删除即将过期的

注意：

LRU 和 LFU 的含义：

		# LRU means Least Recently Used
		# LFU means Least Frequently Used