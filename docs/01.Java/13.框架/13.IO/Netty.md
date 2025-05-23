---
title: Netty 快速入门
date: 2022-02-17 22:34:30
order: 01
categories:
  - Java
  - 框架
  - IO
tags:
  - Java
  - 框架
  - IO
  - Netty
permalink: /pages/f2c7208b/
---

# Netty 快速入门

## Netty 简介

> **Netty 是一款基于 NIO（Nonblocking I/O，非阻塞 IO）开发的网络通信框架**。

### Netty 的特性

- **高并发**：Netty 是一款**基于 NIO**（Nonblocking IO，非阻塞 IO）开发的网络通信框架，对比于 BIO（Blocking I/O，阻塞 IO），他的并发性能得到了很大提高。
- **传输快**：Netty 的传输依赖于**内存零拷贝**特性，尽量减少不必要的内存拷贝，实现了更高效率的传输。
- **封装好**：Netty **封装了 NIO 操作**的很多细节，提供了易于使用调用接口。

## 核心组件

- `Channel`：Netty 网络操作抽象类，它除了包括基本的 I/O 操作，如 bind、connect、read、write 等。
- `EventLoop`：主要是配合 Channel 处理 I/O 操作，用来处理连接的生命周期中所发生的事情。
- `ChannelFuture`：Netty 框架中所有的 I/O 操作都为异步的，因此我们需要 ChannelFuture 的 addListener()注册一个 ChannelFutureListener 监听事件，当操作执行成功或者失败时，监听就会自动触发返回结果。
- `ChannelHandler`：充当了所有处理入站和出站数据的逻辑容器。ChannelHandler 主要用来处理各种事件，这里的事件很广泛，比如可以是连接、数据接收、异常、数据转换等。
- `ChannelPipeline`：为 ChannelHandler 链提供了容器，当 channel 创建时，就会被自动分配到它专属的 ChannelPipeline，这个关联是永久性的。

Netty 有两种发送消息的方式：

- 直接写入 Channel 中，消息从 ChannelPipeline 当中尾部开始移动；
- 写入和 ChannelHandler 绑定的 ChannelHandlerContext 中，消息从 ChannelPipeline 中的下一个 ChannelHandler 中移动。

## 高性能

Netty 高性能表现在哪些方面：

- **NIO 线程模型**：同步非阻塞，用最少的资源做更多的事。
- **内存零拷贝**：尽量减少不必要的内存拷贝，实现了更高效率的传输。
- **内存池设计**：申请的内存可以重用，主要指直接内存。内部实现是用一颗二叉查找树管理内存分配情况。
- **串形化处理读写**：避免使用锁带来的性能开销。
- **高性能序列化协议**：支持 protobuf 等高性能序列化协议。

## 零拷贝

### 传统意义的拷贝

是在发送数据的时候，传统的实现方式是：

`File.read(bytes)`

`Socket.send(bytes)`

这种方式需要四次数据拷贝和四次上下文切换：

1. 数据从磁盘读取到内核的 read buffer

2. 数据从内核缓冲区拷贝到用户缓冲区
3. 数据从用户缓冲区拷贝到内核的 socket buffer
4. 数据从内核的 socket buffer 拷贝到网卡接口（硬件）的缓冲区

### 零拷贝的概念

明显上面的第二步和第三步是非必要的，通过 java 的 FileChannel.transferTo 方法，可以避免上面两次多余的拷贝（当然这需要底层操作系统支持）

- 调用 transferTo，数据从文件由 DMA 引擎拷贝到内核 read buffer
- 接着 DMA 从内核 read buffer 将数据拷贝到网卡接口 buffer

上面的两次操作都不需要 CPU 参与，所以就达到了零拷贝。

### Netty 中的零拷贝

主要体现在三个方面：

**bytebuffer**

Netty 发送和接收消息主要使用 bytebuffer，bytebuffer 使用对外内存（DirectMemory）直接进行 Socket 读写。

原因：如果使用传统的堆内存进行 Socket 读写，JVM 会将堆内存 buffer 拷贝一份到直接内存中然后再写入 socket，多了一次缓冲区的内存拷贝。DirectMemory 中可以直接通过 DMA 发送到网卡接口

**Composite Buffers**

传统的 ByteBuffer，如果需要将两个 ByteBuffer 中的数据组合到一起，我们需要首先创建一个 size=size1+size2 大小的新的数组，然后将两个数组中的数据拷贝到新的数组中。但是使用 Netty 提供的组合 ByteBuf，就可以避免这样的操作，因为 CompositeByteBuf 并没有真正将多个 Buffer 组合起来，而是保存了它们的引用，从而避免了数据的拷贝，实现了零拷贝。

**对于 FileChannel.transferTo 的使用**

Netty 中使用了 FileChannel 的 transferTo 方法，该方法依赖于操作系统实现零拷贝。

## Netty 流程

## 应用

> Netty 是一个广泛使用的 Java 网络编程框架。很多著名软件都使用了它，如：Dubbo、Cassandra、Elasticsearch、Vert.x 等。

有了 Netty，你可以实现自己的 HTTP 服务器，FTP 服务器，UDP 服务器，RPC 服务器，WebSocket 服务器，Redis 的 Proxy 服务器，MySQL 的 Proxy 服务器等等。

```java
public class NettyOioServer {

    public void server(int port) throws Exception {
        final ByteBuf buf = Unpooled.unreleasableBuffer(
                Unpooled.copiedBuffer("Hi!\r\n", Charset.forName("UTF-8")));
        EventLoopGroup group = new OioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();        //1

            b.group(group)                                    //2
             .channel(OioServerSocketChannel.class)
             .localAddress(new InetSocketAddress(port))
             .childHandler(new ChannelInitializer<SocketChannel>() {//3
                 @Override
                 public void initChannel(SocketChannel ch)
                     throws Exception {
                     ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {            //4
                         @Override
                         public void channelActive(ChannelHandlerContext ctx) throws Exception {
                             ctx.writeAndFlush(buf.duplicate()).addListener(ChannelFutureListener.CLOSE);//5
                         }
                     });
                 }
             });
            ChannelFuture f = b.bind().sync();  //6
            f.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully().sync();        //7
        }
    }
}
```

## 参考资料

- **官方**
  - [Netty 官网](https://netty.io/)
  - [Netty Github](https://github.com/netty/netty)
- **文章**
  - [Netty 入门教程——认识 Netty](https://www.jianshu.com/p/b9f3f6a16911)
  - [彻底理解 Netty，这一篇文章就够了](https://juejin.im/post/5bdaf8ea6fb9a0227b02275a)
  - [Java 200+ 面试题补充 ② Netty 模块](https://juejin.im/post/5c81b08f5188257a323f4cef)
