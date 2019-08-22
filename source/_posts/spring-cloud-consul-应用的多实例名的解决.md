---
title: spring cloud consul 应用的多实例名的解决
date: 2019-08-22 09:22:14
categories: spring-cloud
tags:
- consul
---

之前使用eureka时，注册服务的ID 是随机数，eureka上不会出现同一服务多实例的问题。但是，换上了 consul 作为注册中心后，却出现同一个服务拥有多个实例的问题，上次服务挂掉之后的实例还在注册中心上挂着，每次重启多一个实例。

有什么办法去解决这个问题？答案就是自定义spring cloud consul 的注册方法，使其唯一化。

```java
public class ServiceIdRegister extends ConsulServiceRegistry {

    public ServiceIdRegister(ConsulClient client, ConsulDiscoveryProperties properties, TtlScheduler ttlScheduler, HeartbeatProperties heartbeatProperties) {
        super(client, properties, ttlScheduler, heartbeatProperties);
    }

    @Override
    public void register(ConsulRegistration reg) {
        //重新设计id， 服务命-ip-port
        reg.getService().setId(reg.getService().getName() + "-" + reg.getService().getAddress() + "-" + reg.getPort());
        super.register(reg);
    }
}

```

```java
@Configuration
@ConditionalOnConsulEnabled
public class ConsulConfig {

    @Autowired(required = false)
    private TtlScheduler ttlScheduler;

    /**
     * 重写register方法
     *
     * @param consulClient
     * @param properties
     * @param heartbeatProperties
     * @return
     */
    @Bean
    public ServiceIdRegister consulServiceRegistry(ConsulClient consulClient, ConsulDiscoveryProperties properties,
                                                   HeartbeatProperties heartbeatProperties) {
        return new ServiceIdRegister(consulClient, properties, ttlScheduler, heartbeatProperties);
    }

}
```