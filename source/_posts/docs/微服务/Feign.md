---
title: "Feign"
date: 2022-02-11T15:31:01+08:00
categories: 分布式系统
tags:
- feign
---

## 深入Feign 体系架构、底层机制、动态代理、重试

### 构建请求

![](https://gitee.com/littlefxc/oss/raw/master/images/5e12fe9b09c5379227001716-20220211161545138.png)

- ribbon、hystrix：引入feign的同时，ribbon和hystrix这两个组件也会被一同引入。

  - ribbon：利用负载均衡策略选定目标机器
  - hystrix：根据熔断器的开启状态，决定是否发起此次调用

- 动态代理

  Feign是通过一个代理接口进行远程调用，这一步就是为了构造接口的动态代理对象，用来代理远程服务的真实调用，这样你就可以像调用本地方法一样发起HTTP请求，不需要像Ribbon或者Eureka那样在方法调用的地方提供服务名。在Feign中动态代理是通过`Feign.build`返回的构造器来装配相关参数，然后调用`ReflectFeign`的`newInstance`方法创建的。这里就应用到了Builder设计模式。

- Contract

  协议，顾名思义，就像HTTP协议，RPC协议一样，Feign也有自己的一套协议的规范，只不过他解析的不是HTTP请求，而是上一步提到的动态代理类。通过解析动态代理接口+Builder模式，Contract协议会构造复杂的元数据对象MethodMetadata，这里面包含了动态代理接口定义的所有特征。接下来，根据这些元数据生成一系列MethodHandler对象用来处理Request和Response请求。 

  - Contract具有高度可扩展性，可以经由对Contract的扩展，将Feign集成到其他开源组件之中。

### 发起调用

![](https://gitee.com/littlefxc/oss/raw/master/images/5e12fea8099390f030181540.png)

- 拦截器

   拦截器是Spring处理网络请求的经典方案，Feign这里也沿用了这个做法，通过一系列的拦截器对Request和Response对象进行装饰，比如通过RequestInterceptor给Request对象构造请求头。整装待发之后，就是正式发起调用的时候了。

- 发起请求

  又到了Ribbon和Hystrix的出场镜头了。这哼哈二将绝不放过开头和结尾两处重要镜头，正所谓从头到尾都参与了进来。 

  - 重试

    Feign这里借助Ribbon的配置重试器实现了重试操作，可以指定对当前服务节点发起重试，也可以让Feign换一个服务节点重试。

  - 降级

    Feign接口在声明时可以指定Hystrix的降级策略实现类，如果达到了Hystrix的超时判定，或得到了异常结果，将执行指定的降级逻辑。Hystrix降级熔断的内容，将在下一个大章节和大家见面。



## 参考

慕课网

