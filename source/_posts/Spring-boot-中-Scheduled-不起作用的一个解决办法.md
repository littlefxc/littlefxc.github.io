---
title: Spring boot 中 @Scheduled 不起作用的一个解决办法
date: 2019-11-27 09:42:47
categories: springboot
tags:
- 定时任务
---

# Spring boot 中 @Scheduled 不起作用的一个解决办法

在 spring boot 应用中添加定时任务，按照网上的资料却怎么都不能启动，都说是缺少了 `@EnableScheduling`,我在加上了后却任然启动不了。

最后是这样解决的：主要是新增一个 `org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler`  的 bean

```java
@Slf4j
@EnableScheduling
@Service
public class Sched {

    @Autowired
    private StatisticProperties statisticProperties;

    @Autowired
    private JedisCluster jedisCluster;

    @Bean
    public TaskScheduler poolScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setThreadNamePrefix("poolScheduler");
        scheduler.setPoolSize(10);
        return scheduler;
    }

    @Async(value = "asyncPoolTaskExecutor")
    @Scheduled(cron = "*/5 * * * * ?")
    public void clearRealtimeCacheData() {
        log.info("每5秒执行一次");
    }
}
```
