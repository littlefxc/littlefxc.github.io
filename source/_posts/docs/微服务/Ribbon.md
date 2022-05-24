---
title: "Ribbon"
date: 2021-12-31T15:57:21+08:00
categories: 分布式系统
tags:
- ribbon
---


## 负载均衡介绍

将请求或者说流量，以期望的规则分摊到多个操作单元上进行执行。

通过它可以实现横向扩展(scale out)，将冗余的作用发挥为高可用。另外，还可以物尽其用，提升资源使用率。

<!-- more -->

## 概念

### 客户端负载均衡

![image-20211021155325000](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20211021155325000.png)

基于客户端做负载均衡，有一个前提是需要在客户端本地维护一个服务的机器列表，同时在本地指定一个LB策略，然后输出一个服务。服务列表并不是一成不变的，机器列表需要通过注册中心动态更新机器列表。

### 服务端负载均衡

![image-20211021160108842](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20211021160108842.png)

- **大型应用通常是客户端+服务端负载均衡搭配使用**

## 技术选型

![image-20211021160704789](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20211021160704789.png)

红色框表示客户端模式，灰色框表示服务端模式。

客户端模式对开发团队更友好，负载均衡策略直接在代码中，易于直接开发自定义的策略，而服务端模式往往在Nginx等接入层网关中，而Nginx还是比较友好的，如果是F5的话，开发团队根本就没有机会好吧，这也往往是服务端模式的负载均衡运维成本高的原因。

## 深入Ribbon

### 负载均衡策略和原理，加载方式，IPing 机制

![](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/5e12e23709f43f6528861210.png)

一个HttpRequest发过来，先被转发到Eureka上。此时Eureka仍然通过服务发现获取了所有服务节点的物理地址，但问题是他不知道该调用哪一个，只好把请求转到了Ribbon手里。

- IPing

  IPing是Ribbon的一套healthcheck机制，故名思议，就是要Ping一下目标机器看是否还在线，一般情况下IPing并不会主动向服务节点发起healthcheck请求，Ribbon后台通过静默处理返回true默认表示所有服务节点都处于存活状态（和Eureka集成的时候会检查服节点UP状态）。

  IPing 有以下几种方式：

  - DummyPing，默认返回true，即认为所有节点都可用，这也是单独使用Ribbon时的默认模式
  - NIWSDiscoveryPing，借助Eureka服务发现机制获取节点状态，假如节点状态是UP则认为是可用状态
  - PingUrl，它会主动向服务节点发起一次http调用，如果对方有响应则认为节点可用

  第三种方式 PingUrl 比较生猛，会对各个服务节点请求个不停，各个节点有可能扛不住压力，因此除非特殊指定，在和Eureka合作时，一般采用第二种方式。

- IRule

  这就是Ribbon的组件库了，各种负载均衡策略都继承自IRule接口。所有经过Ribbon的请求都会先请示IRule一把，找到负载均衡策略选定的目标机器，然后再把请求转发过去。

###   LoadBalanced 原理解析

![](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/5e12e63009550a2029541170.png)

#### @LoadBalanced

这个注解一头挂在`RestTemplate`上，另一头挂在`LoadBalancerAutoConfiguration`这个类上。它就像连接两个世界的传送门，将所有顶着「LoadBalanced」注解的RestTemplate类，都传入到`LoadBalancerAutoConfiguration`中。如果要深挖底层的作用机制，大家可以发现这个注解的定义上还有一个`@Qualifier`注解。@Qualifier注解搭配@Autowired注解做自动装配，可以通过name属性，将指定的Bean装载到指定位置（即使有两个同样类型的Bean，也可以通过Qualifier定义时声明的name做区分）。这里「LoadBalanced」也是借助Qualifier实现了一个给RestTemplate打标签的功能，凡是被打标的RestTemplate都会被传送到AutoConfig中做进一步改造。

#### LBAutoConfig

从前一步中传送过来的RestTemplate，会经过`LBAutoConfig`的装配，将一系列的`Interceptor`（拦截器）添加到RestTemplate中。拦截器是类似职责链编程模型的结构，我们常见的ServletFilter，权限控制器等，都是类似的模式。Ribbon拦截器会拦截每个网络请求做一番处理，在这个过程中拦截器会找到对应的LoadBalancer对HTTP请求进行接管，接着LoadBalancer就会找到默认或指定的负载均衡策略来对HTTP请求进行转发。

### 负载均衡策略介绍

Ribbon负载均衡的原理是:从EurekaClient类的Bean获取Provider提供者服务列表清单，并且定 期通过IPing类的Bean去判断Provider的可用性。每次RPC到来时，在Provider提供者服务列表中根据 IRule策略类的Bean计算出每次RPC要访问的最终Provider。

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

#### 负载均衡策略配置

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

### Ribbon自定义基于哈希一致性负载均衡策略的IRule

#### 一致性哈希简单介绍

![image-20220211103649256](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220211103649256.png)

说明：

- 上图所示有 4 台服务器，均匀分布在一个环上，
- 提取请求的特征量（可以是请求中的parameter，也可以是整个URL或者请求体中的一个字段），通过某种算法形成摘要，再把摘要通过 hashcode 算法过滤一遍变成一个 int 值，映射到这个环上的某个位置，
- 然后，**顺时针或者逆时针**去寻找离它最近的一个服务器节点。
- 通过一个摘要把这个请求定位到一个圆环上，接下来按照固定方向寻找服务器就行了。
- 假如某个服务节点不可用，那么只需将定位到该节点的请求重新定位到离它最近的节点就好了，其它的请求保持不变，没有任何影响。

#### 自定义IRule实现

```java
/**
 * 基于哈希一致性实现的负载均衡策略
 *
 * @author fengxuechao
 * @date 2022/2/11
 */
public class HashRule extends AbstractLoadBalancerRule {

    @Override
    public void initWithNiwsConfig(IClientConfig iClientConfig) {

    }

    @Override
    public Server choose(Object key) {
        HttpServletRequest request = ((ServletRequestAttributes)
                RequestContextHolder.getRequestAttributes())
                .getRequest();

        String uri = request.getServletPath() + "?" + request.getQueryString();
        return route(uri.hashCode(), getLoadBalancer().getAllServers());
    }

    private Server route(int hashId, List<Server> servers) {
        if (CollectionUtils.isEmpty(servers)) {
            return null;
        }

        // 这个可以换成更好的算法
        TreeMap<Long, Server> serverMap = new TreeMap<>();
        servers.forEach(itemServer -> {
            // 虚化若干个服务节点到环上
            for (int i = 0; i < 8; i++) {
                long hash = hash(itemServer.getId() + i);
                serverMap.put(hash, itemServer);
            }
        });

        long hash = hash(String.valueOf(hashId));
        SortedMap<Long, Server> sortedMap = serverMap.tailMap(hash);

        // request 的 URL 的 hash 值大于任意一个服务器对应的一个 HashKey, 取 servers 中的第一个节点
        if (sortedMap.isEmpty()) {
            return serverMap.firstEntry().getValue();
        }

        return sortedMap.get(sortedMap.firstKey());
    }

    private long hash(String key) {
        MessageDigest md5;
        try {
            md5 = MessageDigest.getInstance("MD5");
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException(e);
        }

        byte[] keyBytes = key.getBytes(StandardCharsets.UTF_8);
        md5.update(keyBytes);
        byte[] digest = md5.digest();

        long hashcode = ((long) (digest[2] & 0xFF << 16))
                | ((long) (digest[1] & 0xFF << 8))
                | ((long) (digest[1] & 0xFF));

        return hashcode & 0xffffffffL;
    }
}
```
