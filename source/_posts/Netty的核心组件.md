---
title: Netty的核心组件
date: 2021-06-09 11:45:56
categories: netty
tags:
- netty
---

# 1 Channel 接口

基本的 I/O 操作(bind()、connect()、read()和 write())依赖于底层网络传输所提 供的原语。在基于 Java 的网络编程中，其基本的构造是 class Socket。Netty 的 Channel 接 口所提供的 API，大大地降低了直接使用 Socket 类的复杂性。此外，Channel 也是拥有许多 预定义的、专门化实现的广泛类层次结构的根，下面是一个简短的部分清单:

<!-- more -->

- EmbeddedChannel; 
- LocalServerChannel; 
- NioDatagramChannel; 
- NioSctpChannel; 
- NioSocketChannel。

# 2 EventLoop 接口

EventLoop 定义了 Netty 的核心抽象，用于处理连接的生命周期中所发生的事件。下图在高层次上说明了 Channel、EventLoop、Thread 以及 EventLoopGroup 之间的关系。

![image-20210621191716357](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210621191716357.png)

这些关系是：

- 一个 EventLoopGroup 包含一个或者多个 EventLoop;

- 一个 EventLoop 在它的生命周期内只和一个 Thread 绑定;

- 所有由 EventLoop 处理的 I/O 事件都将在它专有的 Thread 上被处理;

- 一个 Channel 在它的生命周期内只注册于一个 EventLoop;

- 一个 EventLoop 可能会被分配给一个或多个 Channel。 

**<font color=red>注意</font>**，在这种设计中，一个给定 Channel 的 I/O 操作都是由相同的 Thread 执行的，实际上消除了对于同步的需要。

# 3 ChannelFuture 接口

**Netty 中所有的 I/O 操作都是异步的**。因为一个操作可能不会 立即返回，所以我们需要一种用于在之后的某个时间点确定其结果的方法。为此，Netty 提供了 `ChannelFuture`接口，其`addListener()`方法注册了一个`ChannelFutureListener`，以 便在某个操作完成时(无论是否成功)得到通知。

# 4 ChannelHandler 接口

从应用程序开发人员的角度来看，Netty 的主要组件是 `ChannelHandler`，它充当了所有 处理入站和出站数据的应用程序逻辑的容器。这是可行的，因为` ChannelHandler` 的方法是由网络事件触发的。事实上, `ChannelHandler` 可专 门用于几乎任何类型的动作，例如将数据从一种格式转换为另外一种格式，或者处理转换过程 中所抛出的异常。

举例来说，`ChannelInboundHandler` 是一个你将会经常实现的子接口。这种类型的 `ChannelHandler` 接收入站事件和数据，这些数据随后将会被你的应用程序的业务逻辑所处理。当你要给连接的客户端发送响应时，也可以从 `ChannelInboundHandler` 冲刷数据。你 的应用程序的业务逻辑通常驻留在一个或者多个 `ChannelInboundHandler` 中。

# 5 ChannelPipeline 接口

`ChannelPipeline` 提供了 `ChannelHandler` 链的容器，并定义了用于在该链上传播入站 和出站事件流的 API。当 `Channel` 被创建时，它会被自动地分配到它专属的 `ChannelPipeline`。

`ChannelHandler` 安装到 `ChannelPipeline` 中的过程如下所示: 

- 一个`ChannelInitializer`的实现被注册到了`ServerBootstrap`中;

- 当` ChannelInitializer.initChannel()`方法被调用时，`ChannelInitializer`将在 `ChannelPipeline` 中安装一组自定义的`ChannelHandler`; 
- `ChannelInitializer` 将它自己从 `ChannelPipeline` 中移除。

ChannelHandler 是专为支持广泛的用途而设计的，可以将它看作是处理往来 Channel-

Pipeline 事件(包括数据)的任何代码的通用容器。下图说明了这一点，其展示了从 ChannelHandler 派生的 ChannelInboundHandler 和 ChannelOutboundHandler 接口。

![image-20210621192959857](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210621192959857.png)

使得事件流经 ChannelPipeline 是 ChannelHandler 的工作，它们是在应用程序的初始化或者引导阶段被安装的。这些对象接收事件、执行它们所实现的处理逻辑，并将数据传递给 链中的下一个 ChannelHandler。它们的执行顺序是由它们被添加的顺序所决定的。实际上， 被我们称为 ChannelPipeline 的是这些 ChannelHandler 的编排顺序。

下图说明了一个 Netty 应用程序中入站和出站数据流之间的区别。从一个客户端应用程序 的角度来看，如果事件的运动方向是从客户端到服务器端，那么我们称这些事件为**出站**的，反之则称为**入站**的。

![image-20210621193108690](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/image-20210621193108690.png)

