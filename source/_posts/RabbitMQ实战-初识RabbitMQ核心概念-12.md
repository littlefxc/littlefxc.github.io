---
title: RabbitMQ实战-初识RabbitMQ核心概念(12)
date: 2021-01-25 21:44:01
categories: mq
tags: 
- mq
- rabbitmq
---

# 1 前言

[RabbitMQ（一）：RabbitMQ快速入门](https://www.cnblogs.com/sgh1023/p/11217017.html)

[RabbitMQ基础概念详细介绍](https://www.cnblogs.com/williamjie/p/9481774.html)

![https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_1.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_1.png)

![https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_2.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_2.png)

![https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_3.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_3.png)

![https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_4.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_4.png)

![https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_5.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_5.png)

# 2 AMQP协议中间的几个重要概念：

下图是AMQP的协议模型：

![https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_6.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_6.png)

正如图中所看到的，AMQP协议模型有三部分组成：生产者、消费者和服务端。

生产者是投递消息的一方，首先连接到Server，建立一个连接，开启一个信道；然后生产者声明交换器和队列，设置相关属性，并通过路由键将交换器和队列进行绑定。同理，消费者也需要进行建立连接，开启信道等操作，便于接收消息。

接着生产者就可以发送消息，发送到服务端中的虚拟主机，虚拟主机中的交换器根据路由键选择路由规则，然后发送到不同的消息队列中，这样订阅了消息队列的消费者就可以获取到消息，进行消费。

最后还要关闭信道和连接。

- Server：又称Broker。接收客户端的连接，实现AMQP实体服务。
- Connection：连接，应用程序与Server的网络连接，TCP连接。
- Channel：信道，消息读写（几乎所有操作）等操作在信道中进行。客户端可以建立多个信道，每个信道代表一个会话任务。
- Message：消息，应用程序和服务器之间传送的数据，消息可以非常简单，也可以很复杂。有Properties和Body组成。Properties为外包装，可以对消息进行修饰，比如消息的优先级、延迟等高级特性；Body就是消息体内容。
- Virtual Host：虚拟主机，用于逻辑隔离。一个虚拟主机里面可以有若干个Exchange和Queue，同一个虚拟主机里面不能有相同名称的Exchange或Queue。
- Exchange：交换器，接收消息，按照路由规则将消息路由到一个或者多个队列。如果路由不到，或者返回给生产者，或者直接丢弃。RabbitMQ常用的交换器常用类型有direct、topic、fanout、headers四种，后面详细介绍。
- Binding：绑定，交换器和消息队列之间的虚拟连接，绑定中可以包含一个或者多个RoutingKey。
- RoutingKey：路由键，生产者将消息发送给交换器的时候，会发送一个RoutingKey，用来指定**路由规则**，这样交换器就知道把消息发送到哪个队列。路由键通常为一个“.”分割的字符串，例如“com.rabbitmq”。
- Queue：消息队列，用来保存消息，供消费者消费。

> 我们完全可以直接使用 Connection 就能完成信道的工作，为什么还要引入信道呢?

> 试想这样一个场景， 一个应用程序中有很多个线程需要从 RabbitMQ 中消费消息，或者生产消息，那么必然需要建立很多个 Connection，也就是许多个 TCP 连接。然而对于操作系统而言，建立和销毁 TCP 连接是非常昂贵的开销，如果遇到使用高峰，性能瓶颈也随之显现。 RabbitMQ 采用 TCP 连接复用的方式，不仅可以减少性能开销，同时也便于管理 。

RabbitMQ是基于AMQP协议实现的，其结构如下图所示，和AMQP协议简直就是一模一样。

![https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_7.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_7.png)

![https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_11.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_11.png)

# 3 **常用交换器**

RabbitMQ常用的交换器类型有direct、topic、fanout、headers四种。

## 3.1 Direct Exchange

该类型的交换器将所有发送到该交换器的消息被转发到RoutingKey指定的队列中，也就是说路由到BindingKey和RoutingKey完全匹配的队列中。

![https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_8.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_8.png)

## 3.2 Topic Exchange

该类型的交换器将所有发送到Topic Exchange的消息被转发到所有RoutingKey中指定的Topic的队列上面。

Exchange将RoutingKey和某Topic进行模糊匹配，其中“*”用来匹配一个词，“#”用于匹配一个或者多个词。例如“com.#”能匹配到“com.rabbitmq.oa”和“com.rabbitmq”；而"login.*"只能匹配到“com.rabbitmq”。

![https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_9.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_9.png)

## 3.3 Fanout Exchange

该类型不处理路由键，会把所有发送到交换器的消息路由到所有绑定的队列中。优点是转发消息最快，性能最好。

![https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_10.png](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rabbitmq_11_10.png)

## 3.4 Headers Exchange

该类型的交换器不依赖路由规则来路由消息，而是根据消息内容中的headers属性进行匹配。headers类型交换器性能差，在实际中并不常用。