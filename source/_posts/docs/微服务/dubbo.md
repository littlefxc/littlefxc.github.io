---
title: dubbo
date: 2022-06-06 10:58:28
categories: 分布式系统
tags:
- dubbo
---

# 1 RPC 的第一印象

- 远程方法调用（Remote Procedure Call）
- 服务治理（分布式环境、注册中心）
- RPC协议（方法寻址、对象序列化/反序列化）

# 2 RPC VS HTTP

## 2.1 从接口分格上看

![image-20220606134721000](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20220606134721000.png)

## 2.2 从服务治理上看

|              | RPC                        | HTTP                                   |
| ------------ | -------------------------- | -------------------------------------- |
| 应用层协议   | RPC协议，底层传输基于TCP   | 超文本传输协议，底层传输基于TCP        |
| 编程友好度   | 配置简单高效，接口拿来就用 | 配置繁琐，资源定位，GET/POST……         |
| 传输效率     | 应用gzip等压缩技术         | HTTP携带的信息臃肿报文中有效信息占比小 |
| 框架实现难度 | 难，但和我们没啥关系       | 简单#                                  |

# 3 Dubbo 架构

![](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/5e1437ec0942186f10600824.png)

我们先来认识下Dubbo中的五个基础组件，上图里的紫色线条代表了组件初始化的路径，蓝色虚线是异步通知流程，蓝色实线则是同步阻塞调用。

上图中出现的五个组件分别对应如下功能：

| 组件名称  | 说明                           |
| --------- | ------------------------------ |
| Registry  | 注册中心                       |
| Provider  | 服务提供方                     |
| Consumer  | 向Provider发起远程调用的消费者 |
| Monitor   | 监控中心，用来                 |
| Container | 运行服务的容器                 |

我们弄清楚了谁是谁之后，来捋一捋RPC调用的来龙去脉。我们就顺着图中的步骤标号，从0到5挨个说一下：

- **Start**：服务容器启动后初始化服务提供者
- **Register**：服务提供者在启动的过程中，向注册中心发起注册，这个步骤和Eureka的流程很像
- **Subscribe**： 服务消费者在启动的同时，向注册中心订阅所需的服务。这一个流程就和Eureka就大不相同了，Eureka是Consumer主动到服务中心去拉取数据，而Dubbo采用了一种Pub/Sub模式，也就是发布订阅模型
- **Notify**：注册中心将Provier地址列表发送给消费者，对于服务下线之类的变更，注册中心会主动推送变更数据到Consumer（建立在长连接之上）
- **Invoke**：服务消费者发起远程调用，这个过程会使用负载均衡算法挑选目标服务器
- **Count**：Consumer和Provider每隔一段时间将统计信息发送到监控中心，平时这些信息就暂存于内存当中

# 4 Dubbo和Eureka中服务发现的不同

服务发现应该是Dubbo和Eureka最大的一个不同之处。

Dubbo里的注册中心、Provider和Consumer三者之间都是长连接，借助于Zookeeper的高吞吐量，实现基于服务端的服务发现机制。因此Dubbo利用Zookeeper+发布订阅模型可以很快将服务节点的上线和下线同步到Consumer集群。如果服务提供者宕机，那么注册中心的长连接会立马感知到这个事件，并且立即推送通知到消费者。

在服务发现的做法上Dubbo和Eureka有很大的不同，Eureka使用客户端的服务发现机制，因此对服务列表的变动响应会稍慢，比如在某台机器下线以后，在一段时间内可能还会陆续有服务请求发过来，当然这些请求会收到Service Unavailable的异常，需要借助Ribbon或Hystrix实现重试或者降级措施。

对于注册中心宕机的情况，Dubbo和Eureka的处理方式相同，这两个框架的服务节点都在本地缓存了服务提供者的列表，因此仍然可以发起调用，但服务提供者列表无法被更新，因此可能导致本地缓存的服务状态与实际情况有别。

# 5 Dubbo 核心功能讲解

## 5.1 Registry 模块

Registry是Dubbo中负责服务注册的模块，也就是我们熟悉的注册中心，目前Dubbo支持五种不同的注册中心，其中登场率最高的就是Zookeeper了。

Zookeeper是Apache Hadoop的子项目，它底层是一个树形的目录结构，同时支持基于发布订阅模型的变更推送，这个特性特别适合应用在注册中心服务上。Zookeeper在各个大厂都有广泛应用，其稳定性和性能已经在业界得到了充分证明。大家如果想进一步了解Zookeeper可以逛逛官方开源文档：[http://zookeeper.apache.org](http://zookeeper.apache.org)

### 5.1.1 注册中心的运作方式

注册中心只承担服务治理相关的功能（注册、发现、心跳和下线等），它不承担消息的转发（这一点和Eureka一样），因此不用承担访问压力。和Eureka相比，Dubbo在服务治理领域走的完全是不同的风格路线，我们从各个功能上先来做一个了解

#### Dubbo的长连接

Eureka的注册中心在服务治理领域采用的是一种“佛系”的连接策略，比如服务发现和心跳检测，都是等待服务节点自己上报状态。而Dubbo相比较之下就是“事必躬亲”的管理策略，**Dubbo的注册中心，服务提供者和服务消费者三者之间均为长连接，注册中心严密监测节点的状态变化。**

#### 服务剔除

当服务提供者出现异常情况的时候（比如电源又被挖掘机铲断了），注册中心利用长连接可以侦听到服务的不可用状态，进而在服务列表中删除服务提供者。

我们以Zookeeper注册中心为例，**Dubbo的服务剔除原理实际上是利用Zookeeper的临时节点和会话保持的功能。**ZK中的数据是和一个会话绑定的，一旦会话异常ZK就会清除这个会话创建的临时节点和所有订阅该会话的Watcher。

我们结合Dubbo的长连接和服务剔除机制，讲一讲ZK中的会话（Session）是怎么一回事儿。在ZK中当一个长连接建立起来后，会生成一个Session ID，这是一个全局唯一的标识符。在会话期间内ZK通过心跳感应连接状态，如果在会话过期时间内没有心跳发送过来，或者因为网络原因导致会话断掉，那么这个时候客户端就会重新选择新的ZK节点进行连接。因此，当会话过期时，Dubbo的服务提供者和消费者能自动恢复注册数据，重新订阅请求。

Eureka则不同，**Eureka后台有一个服务自保的机制**，也就是说短期的网络波动导致大比例节点无法连接到注册中心的时候，Eureka会开启自保模式，在这个模式下服务剔除功能将被禁用。

#### 心跳检测

Dubbo同样也有心跳检测，它的**目的是检测服务提供者和服务消费者之间的连接是否还处于可用状态**。如果在默认60s没有消息进来的时候，会尝试发送一个心跳Request，如果连着180秒既没有消息也没心跳，那么则会关闭Channel，对于Consumer来说则需要重新连接一个新的Provider。

#### 监控中心

服务提供者向注册中心注册其提供的服务，并汇报调用时间到监控中心。服务消费者向注册中心获取服务提供者地址列表，并根据负载算法直接调用提供者，同时汇报调用时间到监控中心。

监控中心是一个可选组件，可以根据自身项目需要决定是否接入。监控中心和各个节点之间并不是长连接，同时监控中心的宕机并不会影响任何服务调用，只不过可能会丢失一部分的监控数据而已。

## 5.2 Remoting 模块

Remoting模块主要负责远程调用，核心部分是Dubbo的协议栈实现，处于整个调用链路中相对底层的部分。我们在后面安排了单独一个小节来介绍Dubbo协议的细节，在这里我们简单的看下Remoting框架的接口结构。

![](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/5e143897092e000205620203.png)

### 5.2.1 顶层接口层

- **Endpoint**：它是一个顶层的抽象接口，定义了获取网络端点地址、发送消息、获取通信Channel、关闭连接等一些列方法。
	- **Channel** - 它继承自Endpoint，但是扩展了操作Attribute的方法，比如常用的getAttribute，setAttribute等
- **ChannelHandler**：ChannelHandler定义了一系列接口用来处理五种不同的网络事件，分别对应connected、disconnected、sent、received和caught
- **Resetable**：接口中只定义了一个方法reset，用来重置连接属性

#### AbstractPeer和AbstractChannel

`AbstractPeer`实现了上面提到的两个顶层接口，它是一种代理模式的组合结构，通过构造器接收一个ChannelHandler对象，然后把方法实现的具体逻辑委托给ChannelHandler对象来实现，随手拿一个接收消息的代码来举例：

```java
@Override
public void received(Channel ch, Object msg) throws RemotingException {
    if (closed) {
        return;
    }
    handler.received(ch, msg);
}
```

`AbstractChannel`添加了send方法（发送消息到远程服务）的缺省验证逻辑，作为NettyChannel等具体底层实现类的统一父类

#### Client & Server

`Client`作为服务调用的发起方（消费者）的顶层接口，继承了Endpoint和Channel接口的同时，还定义了reconnect方法，用来重新连接到Server。Dubbo中的底层服务调用实现类（比如NettyClient和MinaClient）都继承自Client，但是Client只是个发起服务调用的壳，底层的调用逻辑都被封装在Channel对象里（AbstractChannel的子类，比如NettyChannel）

`Server`作为服务提供方的顶层接口，继承自Endpoint和Resetable的同时，又定义了一系列服务端的接口方法，比如获取Channel列表或某个指定Channel
