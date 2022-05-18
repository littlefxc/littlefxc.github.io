---
title: 基于Redis的setnx实现分布式锁
date: 2021-05-07 22:46:45
tags:
- 分布式锁
- redis
---

# 基于Redis的setnx实现分布式锁

## 实现原理

- 获取锁的 Redis 命令

  ```sh
  set resource_name my_random_value NX PX 30000
  ```
  
  说明：
   - resource_name：资源名称，可根据不同的业务区分不同的锁
   - my_random_value：随机值，每个线程的随机值都不同，用于释放锁的校验
   - NX：key 不存在时设置成功，key存在则设置不成功
   - PX：自动失效时间，出现异常情况，锁可以过期失效

<!-- more -->

- 释放锁采用Redis的delete命令

- 释放锁时校验之前设置的随机数，相同才能释放（确保只能同一个线程才能释放锁）

- 释放锁的LUA脚本（为什么？因为delete并没有提供校验的功能）

  ![原理](https://gitee.com/littlefxc/oss/raw/master/images/QQ20210507-231010@2x.png)
  
  为了防止出现上面这种情况（A释放了B的锁），于是就有了下面的这段代码，释放锁之前检查一下值正不正确。

  ```lua
  if redis.call("get", KEYS[1]) == ARGV[1] then 
    return redis.call("del", KEYS[1])
  else
    return 0
  end
  ```

## 实战

### 根据上述原理，编写分布式锁

```java
@Slf4j
public class RedisLock implements AutoCloseable {

    private RedisTemplate redisTemplate;
    private String key;
    private String value;
    //单位：秒
    private int expireTime;

    public RedisLock(RedisTemplate redisTemplate,String key,int expireTime){
        this.redisTemplate = redisTemplate;
        this.key = key;
        this.expireTime=expireTime;
        this.value = UUID.randomUUID().toString();
    }

    /**
     * 获取分布式锁
     * @return
     */
    public boolean getLock(){
        RedisCallback<Boolean> redisCallback = connection -> {
            //设置NX
            RedisStringCommands.SetOption setOption = RedisStringCommands.SetOption.ifAbsent();
            //设置过期时间
            Expiration expiration = Expiration.seconds(expireTime);
            //序列化key
            byte[] redisKey = redisTemplate.getKeySerializer().serialize(key);
            //序列化value
            byte[] redisValue = redisTemplate.getValueSerializer().serialize(value);
            //执行setnx操作
            Boolean result = connection.set(redisKey, redisValue, expiration, setOption);
            return result;
        };

        //获取分布式锁
        Boolean lock = (Boolean)redisTemplate.execute(redisCallback);
        return lock;
    }

    public boolean unLock() {
        String script = "if redis.call(\"get\",KEYS[1]) == ARGV[1] then\n" +
                "    return redis.call(\"del\",KEYS[1])\n" +
                "else\n" +
                "    return 0\n" +
                "end";
        RedisScript<Boolean> redisScript = RedisScript.of(script,Boolean.class);
        List<String> keys = Arrays.asList(key);

        Boolean result = (Boolean)redisTemplate.execute(redisScript, keys, value);
        log.info("释放锁的结果："+result);
        return result;
    }


    @Override
    public void close() throws Exception {
        unLock();
    }
}

```

示例：

```java
@RequestMapping("redisLock")
public String redisLock(){
    log.info("我进入了方法！");
    try (RedisLock redisLock = new RedisLock(redisTemplate,"redisKey",30)){
        if (redisLock.getLock()) {
            log.info("我进入了锁！！");
            Thread.sleep(15000);
        }
    } catch (InterruptedException e) {
        e.printStackTrace();
    } catch (Exception e) {
        e.printStackTrace();
    }
    log.info("方法执行完成");
    return "方法执行完成";
}
```

### 通过定时任务（spring-task）集群部署校验编写的分布式锁

![](https://gitee.com/littlefxc/oss/raw/master/images/Springtaskredislock.png)

说明：哪个服务获取锁，就哪个服务执行任务A，来解决任务A重复执行的问题。

```java
@Service
@Slf4j
public class SchedulerService {
    @Autowired
    private RedisTemplate redisTemplate;

    @Scheduled(cron = "0/5 * * * * ?")
    public void sendSms(){
        try(RedisLock redisLock = new RedisLock(redisTemplate,"autoSms",30)) {
            if (redisLock.getLock()){
                log.info("向138xxxxxxxx发送短信！");
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```

