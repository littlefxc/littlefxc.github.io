---
title: zuul超时时间和重试设置
date: 2019-09-11 19:24:48
tags:
---

## Zuul 网关的超时设置

Zuul（网关）的超时时间需要设置zuul、hystrix、ribbon等三部分：

### zuul超时设置

    #zuul超时设置
    #默认1000
    zuul.host.socket-timeout-millis=2000
    #默认2000
    zuul.host.connect-timeout-millis=4000

### hystrix超时设置

    #熔断器启用
    feign.hystrix.enabled=true
    hystrix.command.default.execution.timeout.enabled=true
    #断路器的超时时间,下级服务返回超出熔断器时间，即便成功，消费端消息也是TIMEOUT,所以一般断路器的超时时间需要大于ribbon的超时时间，ribbon是真正去调用下级服务
    #当服务的返回时间大于ribbon的超时时间，会触发重试
    #断路器的超时时间默认为1000ms，太小了
    hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=60000

    # 为某个特定的服务配熔断时间
    hystrix.command.service-a.execution.isolation.thread.timeoutInMilliseconds=60000

    #断路器详细设置
    #当在配置时间窗口内达到此数量的失败后，进行短路。默认20个）
    #hystrix.command.default.circuitBreaker.requestVolumeThreshold=20
    #短路多久以后开始尝试是否恢复，默认5s）
    #hystrix.command.default.circuitBreaker.sleepWindowInMilliseconds=5
    #出错百分比阈值，当达到此阈值后，开始短路。默认50%）
    #hystrix.command.default.circuitBreaker.errorThresholdPercentage=50%

## ribbon超时设置

    #ribbon请求连接的超时时间，限制3秒内必须请求到服务，并不限制服务处理的返回时间
    ribbon.ConnectTimeout=3000

    ribbon.SocketTimeout=5000
    #请求处理的超时时间 下级服务响应最大时间,超出时间消费方（路由也是消费方）返回timeout
    ribbon.ReadTimeout=5000

    # 单独设置某个服务的超时时间，会覆盖其他的超时时间限制，服务的名称以注册中心页面显示的名称为准，超时时间不可大于断路器的超时时间
    service-a.ribbon.ReadTimeout=50000
    service-a.ribbon.ConnectTimeout=50000

## 重试机制

    #重试机制
    #该参数用来开启重试机制，默认是关闭
    spring.cloud.loadbalancer.retry.enabled=true
    #对所有操作请求都进行重试
    ribbon.OkToRetryOnAllOperations=true
    #对当前实例的重试次数
    ribbon.MaxAutoRetries=1
    #切换实例的重试次数
    ribbon.MaxAutoRetriesNextServer=1
    #根据如上配置，当访问到故障请求的时候，它会再尝试访问一次当前实例（次数由MaxAutoRetries配置），
    #如果不行，就换一个实例进行访问，如果还是不行，再换一次实例访问（更换次数由MaxAutoRetriesNextServer配置），
    #如果依然不行，返回失败信息。
