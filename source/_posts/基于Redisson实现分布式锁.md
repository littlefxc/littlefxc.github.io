---
title: 基于Redisson实现分布式锁
date: 2021-05-16 13:39:54
tags:
- 分布式锁
- redis
---

# 前言

Redisson 除了实现了redis基本功能以外，还重新实现了Java并发包里面的内容。

如何使用Redisson？Redisson可以说是redis客户端的加强版本，它里面的内容较多，也提供了分布式锁的实现，使用简单只需简单配置和调用即可，步骤如下：

1. 引入Redisson的Jar包

2. 进行Redisson与Redis的配置

3. 使用分布式锁

使用方式：

- 通过Java API方式引入Redisson
- Spring项目引入Redisson
- SpringBoot项目引入Redisson 

<!-- more -->

# 单元测试

```java
@RunWith(SpringRunner.class)
@SpringBootTest
@Slf4j
public class RedissonLockApplicationTests {

    @Test
    public void contextLoads() {
    }

    @Test
    public void testRedissonLock() {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://localhost");
        RedissonClient redisson = Redisson.create(config);

        RLock rLock = redisson.getLock("order");

        try {
            rLock.lock(30, TimeUnit.SECONDS);
            log.info("我获得了锁！！！");
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            log.info("我释放了锁！！");
            rLock.unlock();
        }
    }

}
```

结果演示：

![](https://gitee.com/littlefxc/oss/raw/master/images/MBFX4f.png)

# 使用示例

引用依赖：

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.15.5</version>
</dependency>
```

配置：

```
spring.redis.host=localhost
```

示例：

```java
@RestController
@Slf4j
public class RedissonLockController {
    @Autowired
    private RedissonClient redisson;

    @RequestMapping("redissonLock")
    public String redissonLock() {
        RLock rLock = redisson.getLock("order");
        log.info("我进入了方法！！");
        try {
            rLock.lock(30, TimeUnit.SECONDS);
            log.info("我获得了锁！！！");
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            log.info("我释放了锁！！");
            rLock.unlock();
        }
        log.info("方法执行完成！！");
        return "方法执行完成！！";
    }
}
```

结果演示：

![](https://gitee.com/littlefxc/oss/raw/master/images/2lvzmZ.png)

![](https://gitee.com/littlefxc/oss/raw/master/images/skDIpq.png)

# 参考资源

[https://github.com/redisson/redisson](https://github.com/redisson/redisson)
