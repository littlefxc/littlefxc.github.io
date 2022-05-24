---
title: 负载均衡Ribbon
date: 2021-10-21 12:01:22
categories: 分布式系统
tags:
- Ribbon
---

# 负载均衡介绍

将请求或者说流量，以期望的规则分摊到多个操作单元上进行执行。

通过它可以实现横向扩展(scale out)，将冗余的作用发挥为高可用。另外，还可以物尽其用，提升资源使用率。

<!-- more -->

## 概念

## 客户端负载均衡

![image-20211021155325000](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20211021155325000.png)

基于客户端做负载均衡，有一个前提是需要在客户端本地维护一个服务的机器列表，同时在本地指定一个LB策略，然后输出一个服务。服务列表并不是一成不变的，机器列表需要通过注册中心动态更新机器列表。

## 服务端负载均衡

![image-20211021160108842](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20211021160108842.png)

- 大型应用通常是客户端+服务端负载均衡搭配使用

## 技术选型

![image-20211021160704789](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20211021160704789.png)

# 深入Ribbon

## 负载均衡策略和原理，加载方式，IPing 机制

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

##   LoadBalanced 原理解析

![](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/5e12e63009550a2029541170.png)

### @LoadBalanced

这个注解一头挂在`RestTemplate`上，另一头挂在`LoadBalancerAutoConfiguration`这个类上。它就像连接两个世界的传送门，将所有顶着「LoadBalanced」注解的RestTemplate类，都传入到`LoadBalancerAutoConfiguration`中。如果要深挖底层的作用机制，大家可以发现这个注解的定义上还有一个`@Qualifier`注解。@Qualifier注解搭配@Autowired注解做自动装配，可以通过name属性，将指定的Bean装载到指定位置（即使有两个同样类型的Bean，也可以通过Qualifier定义时声明的name做区分）。这里「LoadBalanced」也是借助Qualifier实现了一个给RestTemplate打标签的功能，凡是被打标的RestTemplate都会被传送到AutoConfig中做进一步改造。

### LBAutoConfig

从前一步中传送过来的RestTemplate，会经过`LBAutoConfig`的装配，将一系列的`Interceptor`（拦截器）添加到RestTemplate中。拦截器是类似职责链编程模型的结构，我们常见的ServletFilter，权限控制器等，都是类似的模式。Ribbon拦截器会拦截每个网络请求做一番处理，在这个过程中拦截器会找到对应的LoadBalancer对HTTP请求进行接管，接着LoadBalancer就会找到默认或指定的负载均衡策略来对HTTP请求进行转发。