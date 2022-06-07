---
title: Spring Cloud Ribbon 的负载均衡策略
date: 2021-09-28 11:10:17
tags:
- 负载均衡
- ribbon
- java
---

Ribbon负载均衡的原理是:从EurekaClient类的Bean获取Provider提供者服务列表清单，并且定 期通过IPing类的Bean去判断Provider的可用性。每次RPC到来时，在Provider提供者服务列表中根据 IRule策略类的Bean计算出每次RPC要访问的最终Provider。

<!-- more -->

Ribbon内部有一个负载均衡器接口`ILoadBalance`，定义了添加Provider、获取所有的Provider列表、 获取可用的Provider列表等基础的操作。该接口的核心实现类`DynamicServerListLoadBalancer`会通过 EurekaClient(实现类为DiscoveryClient)获取Provider清单，并且通过`IPing`实例定期(每10s)向每 个Provider实例发送“ping”，并且根据Provider是否有响应来判断该Provider提供者实例是否可用。 如果该Provider的可用性发生了改变，或者Provider清单中的数量和之前的不一致，则从注册中心更新或者重新拉取Provider服务实例清单。

每次RPC请求到来时，由Ribbon的IRule负载均衡策略接口的某个实现类就来进行负载均衡。主要的负载均衡的策略实现类如下:

1. 随机策略(RandomRule) 

   该策略实现类从Provider提供者服务列表中随机选择一个Provider服务实例，作为RPC请求的目标Provider。

2. 线性轮询策略(RoundRobinRule)

   RoundRobinRule线性轮询和RandomRule相似，只是每次都取下一个Provider服务器。假设一共有5台Provider服务节点，使用线性轮询策略，第1次取第1台，第2次取第2台，第3次取第3台，以此类推。

3. 响应时间权重策略(WeightedResponseTimeRule)

   WeightedResponseTimeRule策略为每一个Provider服务维护一个权重值，其规则简单概况为 Provider服务响应时间越长，其权重就越小。在进行服务器选择时，权重值越小，被选择的机会越少。 WeightedResponseTimeRule继承了RoundRobinRule，开始时每一个Provider都没有权重值，每当RPC 请求过来时，由其父类的轮询算法完成负载均衡方式。该策略类有一个默认、每30秒执行一次的权 重更新定时任务，该定时任务会根据Provider实例的响应时间更新Provider权重列表。后续有RPC过来时，将根据权重值进行负载均衡。

4. 最少连接策略(BestAvailableRule)

   在进行服务器选择时，该策略类遍历Provider清单，选取出可用的且连接数最少的一个Provider。 该策略类里面有一个LoadBalancerStats类型的成员变量，会存储所有Provider的运行状况和连接数。 在进行负载均衡计算时，如果选取到的Provider为null，那么会调用线性轮询策略重新选取。

5. 重试策略(RetryRule)

   该类会在一定的时限内进行Provider循环重试。RetryRule会在每次选取之后，对选举的Provider进行判断，如果为null或者not alive，会在一定的时限内(如500ms)内会不停的选取和判断。

6. 可用过滤策略(AvailabilityFilteringRule)

   该类扩展了线性轮询策略，会先通过默认的线性轮询策略选取一个Provider，再去判断该Provider 是否超时可用，当前连接数是否超过限制，如果都满足要求，则成功返回。

   简单来说，AvailabilityFilteringRule将对候选的Provider进行可用性过滤，会先过滤掉多次访问 故障而处于断路器跳闸状态的Provider服务，还会过滤掉并发的连接数超过阈值的Provider服务，然后，对剩余的服务列表进行线性轮询。

7. 区域过滤策略(ZoneAvoidanceRule)

   该类扩展了线性轮询策略，除了过滤超时和连接数过多的Provider之外，还会过滤掉不符合要求的Zone区域中的所有节点。

Ribbon实现的负载均衡策略，不止以上7种，还可以实现自定义的策略类。

局部微服务负载均衡配置示例如下：

```yaml
user-provider:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RetryRule #重试+线性轮询
  # NFLoadBalancerRuleClassName: com.netflix.loadbalancer.BestAvailableRule #最少连接策略
  # NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule #随机选择
```

如果要配置全局的、针对所有的Provider都使用的负载均衡策略，可以在配置文件中直接使用 `ribbon.NFLoadBalancerRuleClassName`配置项进行配置，具体如下:

```yaml
ribbon:
  NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RetryRule #重试+线性轮询
# NFLoadBalancerRuleClassName: com.netflix.loadbalancer.BestAvailableRule #最少连接策略
# NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule #随机选择
```

