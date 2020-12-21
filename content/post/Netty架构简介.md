---
title: "Netty架构简介"
date: 2020-11-24T17:28:06+08:00
description: "Article description."
draft: false
toc: TRUE
categories:
  - Netty
tags:
  - 高性能组件
  - 代码研究
---

Netty功能特性如下

1）传输服务：支持 BIO 和 NIO；

2）容器集成：支持 OSGI、JBossMC、Spring、Guice 容器；

3）协议支持：HTTP、Protobuf、二进制、文本、WebSocket 等一系列常见协议都支持。还支持通过实行编码解码逻辑来实现自定义协议；

4）Core 核心：可扩展事件模型、通用通信 API、支持零拷贝的 ByteBuf 缓冲对象。

![](https://gitee.com/1162492411/pic/raw/master/Netty功能特新.png)



## 高性能设计

Netty 作为异步事件驱动的网络，高性能之处主要来自于其 I/O 模型和线程处理模型，前者决定如何收发数据，后者决定如何处理数据

Netty采用的I/O模型为NIO,如下图

![image-20201124173006255](https://gitee.com/1162492411/pic/raw/master/Netty-Nio.png)



Netty采用的线程处理模型为Reactor模型.Reactor 模型中有 2 个关键组成：

1）Reactor：Reactor 在一个单独的线程中运行，负责监听和分发事件，分发给适当的处理程序来对 IO 事件做出反应。它就像公司的电话接线员，它接听来自客户的电话并将线路转移到适当的联系人；

2）Handlers：处理程序执行 I/O 事件要完成的实际事件，类似于客户想要与之交谈的公司中的实际官员。Reactor 通过调度适当的处理程序来响应 I/O 事件，处理程序执行非阻塞操作。

![image-20201124173023366](https://gitee.com/1162492411/pic/raw/master/Netty-Reactor模型.png)

Reactor模型共有3个变种:单 Reactor 单线程、单 Reactor 多线程、主从 Reactor 多线程.

![image-20201124173254375](https://gitee.com/1162492411/pic/raw/master/Netty-Reactor.png)

![image-20201124173306834](https://gitee.com/1162492411/pic/raw/master/Netty-Reactor多线程.png)

Netty的线程模型是基于主从 Reactors 多线程模型进行修改.

## 核心组件

**Boostrap**:客户端程序的启动引导类,主要作用是配置整个 Netty 程序，串联各个组件

**ServerBootstrap**:服务端启动引导类

**ChannelEvent** : 因为Netty是基于事件驱动的，ChannelEvent就相当于某一个事件，比如说连接成功时打印一句话

**Channel**:网络通信的组件，能够用于执行网络 I/O 操作,下面是一些常用的 Channel 类型：

> NioSocketChannel，异步的客户端 TCP Socket 连接。 NioServerSocketChannel，异步的服务器端 TCP Socket 连接。 NioDatagramChannel，异步的 UDP 连接。 NioSctpChannel，异步的客户端 Sctp 连接。 NioSctpServerChannel，异步的 Sctp 服务器端连接，这些通道涵盖了 UDP 和 TCP 网络 IO 以及文件 IO。

**Selector**:通过 Selector 一个线程可以监听多个连接的 Channel 事件。当向一个 Selector 中注册 Channel 后，Selector 内部的机制就可以自动不断地查询Selector中注册的 Channel 是否有已就绪的 I/O 事件（例如可读，可写，网络连接完成等），这样程序就可以很简单地使用一个线程高效地管理多个 Channel

**NioEventLoop**:NioEventLoop 中维护了一个线程和任务队列，支持异步提交执行任务，线程启动时会调用 NioEventLoop 的 run 方法，执行 I/O 任务和非 I/O 任务

**NioEventLoopGroup** : 主要管理 eventLoop 的生命周期，可以理解为一个线程池，内部维护了一组线程，每个线程(NioEventLoop)负责处理多个 Channel 上的事件

**ChannelHandler** : 一个接口，处理 I/O 事件或拦截 I/O 操作，并将其转发到其 ChannelPipeline(业务处理链)中的下一个处理程序。

ChannelHandler 本身并没有提供很多方法，因为这个接口有许多的方法需要实现，方便使用期间，可以继承它的子类：

> ChannelInboundHandler 用于处理入站 I/O 事件。 ChannelOutboundHandler 用于处理出站 I/O 操作。

或者使用以下适配器类：

> ChannelInboundHandlerAdapter 用于处理入站 I/O 事件。 ChannelOutboundHandlerAdapter 用于处理出站 I/O 操作。 ChannelDuplexHandler 用于处理入站和出站事件。

**ChannelPipline** : 保存 ChannelHandler 的 List，用于处理或拦截 Channel 的入站事件和出站操作,可以理解为一种高级形式的拦截过滤器模式

**ChannelHandlerContext** : 保存 Channel 相关的所有上下文信息



## 组件间关系

当客户端和服务端连接的时候会建立一个 Channel,这个 Channel 我们可以理解为 Socket 连接，它负责基本的 IO 操作，例如：bind（），connect（），read（），write（） 等等,简单的说，Channel 就是代表连接，实体之间的连接，程序之间的连接，文件之间的连接，设备之间的连接。同时它也是数据入站和出站的载体。

EventLoopGroup、EventLoop、Channel关系如下

![image-20201124173323283](https://gitee.com/1162492411/pic/raw/master/Netty-EventLoopGroup、EventLoop、Channel关系.png)

在 Netty 中每个 Channel 都有且仅有一个 ChannelPipeline 与之对应，它们的组成关系如下：

![image-20201124173335104](https://gitee.com/1162492411/pic/raw/master/Netty-Channel与ChannelPipeline关系.png)

一个 Channel 包含了一个 ChannelPipeline，而 ChannelPipeline 中又维护了一个由 ChannelHandlerContext 组成的双向链表，并且每个 ChannelHandlerContext 中又关联着一个 ChannelHandler。

入站事件和出站事件在一个双向链表中，入站事件会从链表 head 往后传递到最后一个入站的 handler，出站事件会从链表 tail 往前传递到最前一个出站的 handler，两种类型的 handler 互不干扰。

这些核心组件的整体关系如下

![image-20201124173347334](https://gitee.com/1162492411/pic/raw/master/Netty-核心组件整体关系.png)

## 核心工作流程

**典型的初始化并启动 Netty 服务端的过程代码如下：**



```java
public final class EchoServer {

    static final boolean SSL = System.getProperty("ssl") != null;
    static final int PORT = Integer.parseInt(System.getProperty("port", "8007"));

    public static void main(String[] args) throws Exception {
        // 配置SSL
        final SslContext sslCtx;
        if (SSL) {
            SelfSignedCertificate ssc = new SelfSignedCertificate();
            sslCtx = SslContextBuilder.forServer(ssc.certificate(), ssc.privateKey()).build();
        } else {
            sslCtx = null;
        }
        // 配置服务端
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        final EchoServerHandler serverHandler = new EchoServerHandler();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class)
             .option(ChannelOption.SO_BACKLOG, 100)
             .handler(new LoggingHandler(LogLevel.INFO))
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     if (sslCtx != null) {
                         p.addLast(sslCtx.newHandler(ch.alloc()));
                     }
                     //p.addLast(new LoggingHandler(LogLevel.INFO));
                     p.addLast(serverHandler);
                 }
             });

            // Start the server.
            ChannelFuture f = b.bind(PORT).sync();

            // Wait until the server socket is closed.
            f.channel().closeFuture().sync();
        } finally {
            // Shut down all event loops to terminate all threads.
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```



**基本过程描述如下：**

1）初始化创建 2 个 NioEventLoopGroup：其中 boosGroup 用于 Accetpt 连接建立事件并分发请求，workerGroup 用于处理 I/O 读写事件和业务逻辑。

2）基于 ServerBootstrap(服务端启动引导类)：配置 EventLoopGroup、Channel 类型，连接参数、配置入站、出站事件 handler。

3）绑定端口：开始工作。



Netty启动流程图如下

![image-20201124173402409](https://gitee.com/1162492411/pic/raw/master/Netty-启动流程.png)

**结合上面介绍的 Netty Reactor 模型，介绍服务端 Netty 的工作架构图：**

![image-20201124173417511](https://gitee.com/1162492411/pic/raw/master/Netty-工作架构图.png)

ps:上图中NioEventGroup有误,应为NioEventLoop

