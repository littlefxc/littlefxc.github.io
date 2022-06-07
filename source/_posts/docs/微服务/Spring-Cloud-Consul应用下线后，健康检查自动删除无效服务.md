---
title: Spring Cloud Consul应用下线后，健康检查自动删除无效服务
date: 2019-08-22 09:19:24
categories: spring-cloud
tags: 
- consul
---

```yaml
spring:
  cloud:
    consul:
      discovery:
        # 健康检查失败多长时间后，取消注册
        health-check-critical-timeout: 30s
```

在配置文件中如上配置后可以使得服务下线后自动删除无效服务，而不必像很多的博客中写的那样专门写一个删除失效服务。

其它的配置属性解析：

- spring.cloud.consul.host：配置consul地址
- spring.cloud.consul.port：配置consul端口
- spring.cloud.consul.discovery.enabled：启用服务发现
- spring.cloud.consul.discovery.register：启用服务注册
- spring.cloud.consul.discovery.deregister：服务停止时取消注册
- spring.cloud.consul.discovery.prefer-ip-address：表示注册时使用IP而不是hostname
- spring.cloud.consul.discovery.health-check-interval：健康检查频率
- spring.cloud.consul.discovery.health-check-path：健康检查路径
- spring.cloud.consul.discovery.health-check-critical-timeout：健康检查失败多长时间后，取消注册
- spring.cloud.consul.discovery.instance-id：服务注册标识