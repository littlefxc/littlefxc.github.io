---
title: netty入门示例
date: 2021-05-11 16:13:49
categories: netty
tags: 
- netty
---

# 前言

现在，我们开始编写一个最简单的Netty示例，在这之前我们先熟悉一下最基本的编码实现步骤！

Netty实现通信的步骤：（客户端与服务器端基本一致）

- 创建两个的NIO线程组，一个专门用于网络事件处理（接受客户端的连接），另一个则进行网络通信读写。
- 创建一个ServerBootstrap对象，配置Netty的一系列参数，例如接受传出数据的缓存大小等等。
- 创建一个实际处理数据的类ChannelInitializer，进行初始化的准备工作，比如设置接受传出数据的字符集、格式、已经实际处理数据的接口。
- 绑定端口，执行同步阻塞方法等待服务器端启动即可。

<!-- more -->

Netty的使用非常简单，仅仅引入依赖即可快速开始：

```xml
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.12.Final</version>
</dependency>
```

#  Netty Server

Netty Server端需要编写 Server 与 ServerHandler两个核心类！

## Server

```java
public class Server {

    public static void main(String[] args) throws InterruptedException {

        //1. 创建两个线程组: 一个用于进行网络连接接受的 另一个用于我们的实际处理（网络通信的读写）

        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workGroup = new NioEventLoopGroup();

        //2. 通过辅助类去构造server/client
        ServerBootstrap b = new ServerBootstrap();

        //3. 进行Nio Server的基础配置

        //3.1 绑定两个线程组
        b.group(bossGroup, workGroup)
                //3.2 因为是server端，所以需要配置NioServerSocketChannel
                .channel(NioServerSocketChannel.class)
                //3.3 设置链接超时时间
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
                //3.4 设置TCP backlog参数 = sync队列 + accept队列
                .option(ChannelOption.SO_BACKLOG, 1024)
                //3.5 设置配置项 通信不延迟
                .childOption(ChannelOption.TCP_NODELAY, true)
                //3.6 设置配置项 接收与发送缓存区大小
                .childOption(ChannelOption.SO_RCVBUF, 1024 * 32)
                .childOption(ChannelOption.SO_SNDBUF, 1024 * 32)
                //3.7 进行初始化 ChannelInitializer , 用于构建双向链表 "pipeline" 添加业务handler处理
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        //3.8 这里仅仅只是添加一个业务处理器：ServerHandler（后面我们要针对他进行编码）
                        ch.pipeline().addLast(new ServerHandler());
                    }
                });

        //4. 服务器端绑定端口并启动服务;使用channel级别的监听close端口 阻塞的方式
        ChannelFuture cf = b.bind(8765).sync();
        cf.channel().closeFuture().sync();

        //5. 释放资源
        bossGroup.shutdownGracefully();
        workGroup.shutdownGracefully();

    }
}
```

## ServerHandler

```java
public class ServerHandler extends ChannelInboundHandlerAdapter {

    /**
     * channelActive
     * 通道激活方法
     */
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        System.err.println("server channel active..");
    }

    /**
     * channelRead
     * 读写数据核心方法
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {

        //1. 读取客户端的数据(缓存中去取并打印到控制台)
        ByteBuf buf = (ByteBuf) msg;
        byte[] request = new byte[buf.readableBytes()];
        buf.readBytes(request);
        String requestBody = new String(request, "utf-8");
        System.err.println("Server: " + requestBody);

        //2. 返回响应数据
        String responseBody = "返回响应数据，" + requestBody;
        ctx.writeAndFlush(Unpooled.copiedBuffer(responseBody.getBytes()));

    }

    /**
     * exceptionCaught
     * 捕获异常方法
     */
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.fireExceptionCaught(cause);
    }

}
```

# Netty Client

Netty Client端需要编写 Client与 ClientHandler两个核心类！

## Client

```java
public class Client {
	public static void main(String[] args) throws InterruptedException {
		
		//1. 创建两个线程组: 只需要一个线程组用于我们的实际处理（网络通信的读写）
		EventLoopGroup workGroup = new NioEventLoopGroup();
		
		//2. 通过辅助类去构造client,然后进行配置响应的配置参数
		Bootstrap b = new Bootstrap();
		b.group(workGroup)
		 .channel(NioSocketChannel.class)
		 .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
		 .option(ChannelOption.SO_RCVBUF, 1024 * 32)
		 .option(ChannelOption.SO_SNDBUF, 1024 * 32)
		 //3. 初始化ChannelInitializer
		 .handler(new ChannelInitializer<SocketChannel>() {
			@Override
			protected void initChannel(SocketChannel ch) throws Exception {
				//3.1  添加客户端业务处理类
				ch.pipeline().addLast(new ClientHandler());	
			}
		});
		//4. 服务器端绑定端口并启动服务; 使用channel级别的监听close端口 阻塞的方式
		ChannelFuture cf = b.connect("127.0.0.1", 8765).syncUninterruptibly();

		//5. 发送一条数据到服务器端
		cf.channel().writeAndFlush(Unpooled.copiedBuffer("hello netty!".getBytes()));
		
		//6. 休眠一秒钟后再发送一条数据到服务端
		Thread.sleep(1000);
		cf.channel().writeAndFlush(Unpooled.copiedBuffer("hello netty again!".getBytes()));

		//7. 同步阻塞关闭监听并释放资源
		cf.channel().closeFuture().sync();
		workGroup.shutdownGracefully();
		
	}
}
```

## ClientHandler

```java
public class ClientHandler extends ChannelInboundHandlerAdapter {
    
    /**
     *  channelActive
     *  客户端通道激活
     */
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
    	System.err.println("client channel active..");
    }
    
    /**
     *  channelRead
     *  真正的数据最终会走到这个方法进行处理
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    	
    	// 固定模式的 try .. finally  
    	// 在try代码片段处理逻辑, finally进行释放缓存资源, 也就是 Object msg (buffer)
        try {
            ByteBuf buf = (ByteBuf) msg;
            byte[] req = new byte[buf.readableBytes()];
            buf.readBytes(req);

            String body = new String(req, "utf-8");
            System.out.println("Client :" + body );
            String response = "收到服务器端的返回信息：" + body;
        } finally {
            ReferenceCountUtil.release(msg);
        }
    }

    /**
     *  exceptionCaught
     *  异常捕获方法
     */
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.fireExceptionCaught(cause);
    }
}
```

# 结果

![vYiVSQ.png](https://gitee.com/littlefxc/oss/raw/master/images/vYiVSQ.png)

客户端向服务端发送数据

![eUQopw.png](https://gitee.com/littlefxc/oss/raw/master/images/eUQopw.png)

客户端收到服务端返回的数据