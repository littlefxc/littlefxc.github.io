---
title: "Gateway"
date: 2022-04-20T15:31:01+08:00
categories: 分布式系统
tags:
- 网关
---

## 1 Spring Cloud Gateway 能做什么？

![image-20220420203439069](https://img-blog.csdnimg.cn/img_convert/45680ad1cd1751edfb2ccbbe1c2eb829.png)

1. 路由寻址（主要功能）
2. 负载均衡
3. 限流
4. 鉴权

<!-- more -->

## 2 Gateway VS Zuul

|               | Gateway  | Zuul 1.x         | Zuul 2.x     |
| ------------- | -------- | ---------------- | ------------ |
| 靠谱性        | 官方支持 | 曾经靠谱过       | 专业放鸽子   |
| 性能          | Netty    | 同步阻塞、性能慢 | Netty        |
| RPS           | > 32000  | 20000 左右       | 25000 左右   |
| Spring Cloud  | 已整合   | 已整合           | 暂无整合计划 |
| 长连接        | 支持     | 不支持           | 支持         |
| 编程体验      | 略难     | 简单易上手       | 略难         |
| 调试&链路追踪 | 略难     | 无压力           | 略难         |

## 3 Gateway 体系架构解析

Netty 在 Gateway 中主要应用在以下几个地方：

- 发起服务调用：由 NettyRoutingFilter 过滤器实现，底层采用基于 Netty 的 HttpClient 发起外部调用
- Response 传输：由 NettyResponseFilter 过滤器实现，网络请求结束后要将 Response 回传给调用者
- Socket 连接：具体由 ReactorNettyWebsocketClient 类继承，通过 Netty 的 HttpClient 发起连接请求

在 Gateway 中发起 Request 和回传 Response 之类的步骤都是通过一系列过滤器完成的，有关过滤器的内容在后文稍后介绍。

Netty 在 Gateway 组件中的位置如下：

![5e13f0b9099a0d7930041532.png (3004×1532) (mukewang.com)](https://img-blog.csdnimg.cn/img_convert/f3feeb8bd2d96787d529875fddb5be63.png)

Client发起请求到服务网关之后，由 NettyRoutingFilter 底层的 HttpClient（也是Netty组件）向服务发起调用，调用结束后的Response由NettyResponseFilter再回传给客户端。有了Netty的加持，网络请求效率大幅提升（Zuul 1.x还是使用Servlet，在2.x版本才移步到Netty）由此可见，Netty贯穿了从Request发起到Response结束的过程，承担了所有和网络调用相关的任务。

### 3.1 Gateway 自动装配

那么Gateway 自动装配需要加载哪些资源呢？

![](https://img-blog.csdnimg.cn/img_convert/f157b5764fb4d16c784ed4bc13f3e398.png)

- AutoConfig：核心自动装配主类，GatewayAutoConfiguration 负责初始化所有的 Route 路由规则、Predicate 断言工厂和 Filter (包括 Global Filter 和 Route Filter)，这三样是 Gateway 吃饭的家伙，用来完成路由功能。AutoConfig 也会同时加载 Netty 配置；
- LoadBalanceClient：这部分在 AutoConfig 完成之后由 GatewayLoadBalanceClientAutoConfiguration 负责加载，用来加载 Ribbon 和一系列负载均衡配置；
- ClassPathWarning：同样也是在 AutoConfig 完成之后触发（具体加载类为 GatewayClassPathWarningAutoConfiguration），由于 Gateway 底层依赖 Spring WebFlux 的实现，所以它会检查项目是否加载了正确配置；
- Redis：在Gateway 中主要负责限流的功能。

除了上面的核心装配工厂以外，还有其他的一些功能：

- GatewayMetricsAutoConfiguration 负责做一些统计工作，比如对所谓的 short task 运行时长和调用次数做统计。
- GatewayDiscoveryClientAutoConfiguration 服务发现客户端自动装配类

## 4 路由功能详解

Gateway 中可以定义多个 Route，一个 Route 就是一套包含完整转发规则的路由，主要由三部分组成：

![](https://img-blog.csdnimg.cn/img_convert/b895129eee8c7e9884625ff8ec753f9a.png)

- **断言集合：**断言是路由处理的第一个环节，它是路由的匹配规则，它决定了一个网络请求是否可以匹配到当前路由来处理。之所以它是一个集合的原因是我们可以给一个路由添加多个断言，当每个断言都匹配成功之后才算过了路由的第一步。
- **过滤器集合：**如果请求通过了前面的断言匹配，那就表示该请求正式被当前路由处理，接下来这个请求需要经过一系列的过滤器集合。而过滤器的功能就是八仙过海各显神通了，可以对当前请求做一系列的操作，比如权限验证，或者将其它非业务性校验的规则提取到网关过滤器这一层。在过滤器这层依然可以对请求做一些额外操作，例如可以修改 Response 里的 Status Code 达到中断效果，如把鉴权失败的请求中的 Status Code设置为 403，然后中断操作。
- **URI：**如果请求顺利地通过了过滤器集合，接下来的就是转发请求了。URI 是统一资源标识符，它可以是一个具体的网址，也可以是 IP + 端口的组合，或者可以是注册中心的服务名。

### 4.1 路由的工作流程

![](https://img-blog.csdnimg.cn/img_convert/5caf80b8df7d33ffdd1e9c940737c89d.png)

- Predicate Handler

  具体实现类是`RoutePredicateHandlerMapping`。首先它获取所有的路由（配置的 routes 全集），然后依次循环每个 Route，把应用请求与 Route 中配置的所有断言进行匹配，如果当前 Route 所有断言都验证通过，Predicate Handler 就选定当前路由。这个设计模式是职责链设计模式。

- Filter Handler

  在前一步选中路由后，由 FilterWebHandler 将请求交给过滤器，在具体处理过程中， 不仅当前Route中定义的过滤器会生效，在项目中添加的全局过滤器（Global Filter）也会一同参与。

  Pre Filter 和 Post Filter 是过滤器的应用阶段，稍后介绍。

- 寻址

  这一步将请求转发到 URI 指定的路径，在发送请求之前，所有 Pre 类型的过滤器都将被执行，而 Post 过滤器会在调用请求返回之后起作用。

## 5 断言功能详解

### 5.1 Predicate 机制

Predicate 是 Java 8 中引入的一个新功能，就和我们平时在项目中写单元测试时用到的 Assertion 差不多，Predicate 接受一个判断条件，返回一个 true 或 false 的布尔值结果，告知调用方判断结果。你也可以通过“与”、“或”和“非”三个操作符将多个 Predicate 串联在一起共同判断。

Predicate 就是一种路由规则，通过 Gateway 中丰富的内置断言的组合，我们就能让一个请求找到对应的 Route 来处理。

### 5.2 断言的作用阶段

一个请求进入网关之后，首先要先进行断言匹配，在满足所有断言之后才会进入 Filter 阶段。有关 Filter 的内容将在之后介绍，本小节不做介绍。

### 5.3 常用断言介绍

#### 5.3.1 Path 断言

```java
.route(r -> r.path("/gateway/**")
             .uri("lb://FEIGN-SERVICE-PROVIDER/")
)
.route(r -> r.path("/baidu")
             .uri("http://baidu.com:80/")
)
```

Path 断言的使用非常简单，就像我们在 Controller 中配置 `@RequestMapping` 的方式一样，在 Path 断言中填上一段 URL 匹配规则，当实际请求的 URL 和断言中的规则相匹配的时候，就下发到该路由中 URI 指定的地址，这个地址可以是一个具体的 HTTP 地址，也可以是注册中心中注册的服务名称。

#### 5.3.2 Method 断言

这个断言是专门验证 HTTP Method 的，在下面的例子中，我们把 Method 断言和 Path 断言通过一个 and 连接符合并起来，共同作用于路由判断，当我们访问“/gateway/sample”并且 HTTP Method 是 GET 的时候，将适配下面的路由。

```java
.route(r -> r.path("/gateway/**")
             .and().method(HttpMethod.GET)
             .uri("lb://FEIGN-SERVICE-PROVIDER/")
)
```

#### 5.3.3 RequestParam匹配

请求断言也是在业务中经常使用的，它会从ServerHttpRequest中的Parameters列表中查询指定的属性，有如下两种不同的使用方式。

```java
.route(r -> r.path("/gateway/**")
             .and().method(HttpMethod.GET)
             .and().query("name", "test")
             .and().query("age")
             .uri("lb://FEIGN-SERVICE-PROVIDER/")
)
```

- 属性名验证

  如`query("age")`，此时断言只会验证QueryPrameters列表中是否包含了一个叫age的属性，并不会验证它的值。

- 属性值验证

  如`query("name", "test")`，它不仅会验证name属性是否存在，还会验证它的值是不是和断言相匹配，比如当前的断言会验证请求参数中的name属性值是不是test，第二个参数实际上是一个用作模式匹配的正则表达式。

#### 5.3.4 Header 断言

这个断言会检查Header中是否包含了响应的属性，通常可以用来验证请求是否携带了访问令牌，比如如下设置：

```java
.route(r -> r.path("/gateway/**")
             .and().header("Authorization")
             .uri("lb://FEIGN-SERVICE-PROVIDER/")
)
```

上面的断言指定了Header中必须包含一个`Authorization`属性，Header断言和Query断言一样，也可以通过传入两个参数的形式对属性值进行检查

#### 5.3.5 Cookie 断言

顾名思义，Cookie验证的是Cookie中保存的信息，Cookie断言和上面介绍的两种断言使用方式大同小异，唯一的不同是它必须连同属性值一同验证，不能单独只验证属性是否存在，示例如下：

```java
.route(r -> r.path("/gateway/**")
             .and().cookie("name", "test")
             .uri("lb://FEIGN-SERVICE-PROVIDER/")
)
```

#### 5.3.6 时间片匹配

时间匹配有三种模式，分别是Before、After和Between，这些断言指定了在什么时间范围内路由才会生效

```java
.route(r -> r.path("/gateway/**")
             .and().before(ZonedDateTime.now().plusMinutes(1))
             .uri("lb://FEIGN-SERVICE-PROVIDER/")
)
```

以Before断言为例，它接受的是一个ZonedDateTime参数，用来表示生效的时间。比如上面的例子中我们使用了`ZonedDateTime.now().plusMinutes(1)`表示当前时间的后一分钟，由于路由的规则是在项目启动时加载的，那么这里的当前时间也就是项目加载完成的时间，因此该路由的有效时间就是服务启动后的一分钟以内。

#### 5.3.7 自定义断言

Gateway也提供了一个扩展方法，用来将自定义的断言应用到路由上。

-  提示1：所有断言类都可以继承自`AbstractRoutePredicateFactory`
-  提示2：在路由配置时可以通过`predicate`或者传入一个自定义断言

## 6 过滤器的原理和生命周期

### 6.1 过滤器的设计模式

其实所有的开源框架的实现过滤器的模式都是大同小异，通过一种类似职责链的方式，传统的职责链模式中的事件会传递直到有一个处理对象接手，Gateway 中的过滤器经过优先级的排列，所有网关请求从最高优先级的过滤器开始，一路走到底，直到被最后一个过滤器处理。

### 6.2 过滤器的实现方式

在 Gateway 中实现一个过滤器非常简单，只要实现 GatewayFilter 接口的默认方法就好了。

```java
@Override
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    // 随意发挥
    return chain.filter(exchange);    
}
```

这里面有两个关键信息

- ServerWebExchange

  这是 Spring 封装的 HTTP request-response 交互协议，从中我们可以获取 request 和 response 中的各种请求参数，也可以向其中添加内容

- GatewayFilterChain

  它是过滤器的调用链，在方法结束的时候我们需要将 exchange 对象传入调用链中的下一个对象。

### 6.3 过滤器的执行阶段 

不同于 Zuul 中对过滤器的 Pre 和 Post 的定义，Gateway 是通过 Filter 中的代码来实现类似 Post 和 Pre 的效果。

Pre 和 Post 是值当前过滤器的执行阶段，Pre 是在下一个过滤器之前被执行，Post 是在过滤器执行过后在执行。我们也可以在过滤器中同时定义 Pre 和 Post 的执行逻辑。

#### 6.3.1 Pre 类型

我们就拿 AddResponseHeaderGatewayFilterFactory 举例，它可以向 Response 中添加 Header 信息：

```java
@Override
public GatewayFilter apply(NameValueConfig config) {
	return (exchange, chain) -> {
        exchange.getResponse().getHeaders().add(config.getName(), config.getValue());
        return chain.filter(exchange);
    };
}
```

这里的具体执行方法是定义在调用`chain.filter()`方法之前，也就是在转发到下级调用链路之前执行的，因此可以理解为一个 Pre 类型的过滤器。

#### 6.3.2 Post 类型

我们拿 SetStatusGatewayFilterFactory 举例，它在过滤器执行完毕之后，将制定的 HTTP status 返回给调用方。

```java
return chain.filter(exchange).then(Mono.fromRunnable(() -> {
		// 这里是业务逻辑
		}));
```

这个过滤器的主要逻辑在 then 方法中，then 是一个回调函数，在下级调用链路都完成以后在执行，因此这类过滤器可以看作是 Post Filter。

### 6.4 过滤器优先级

在 Gateway 中，我们可以通过实现 `org.springframework.core.Ordered`接口或添加`@Order`注解，来给过滤器指定执行顺序，如下所示：

```java
@Override
public int getOrder() {
	return 0;
}
```

