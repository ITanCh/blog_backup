title: Netty入门-架构
date: 2017-05-30 20:48:48
tags: [Netty,java,技术]
---

> 循环加消息队列是实现异步和事件驱动的有效方式。在Android框架和Nodejs框架中，都采用了类似的架构。

这篇文章主要介绍

## 整体架构

![](http://oq1pehpfd.bkt.clouddn.com/blog/netty%E7%BB%93%E6%9E%84.png-blog)

1. 一个EventLoopGroup包含一个或者多个EventLoop；   
2. 一个EventLoop在它的生命周期内只和一个Thread绑定；   
3. 一个Channel在生命周期内只和一个EventLoop绑定；
4. EventLoop可以被分配给多个Channel；
5. 一个Channel有一个ChannelPipeline；   
6. ChannelPipeline中有双向的入站/出站的Channelhandler链。

## 基于NIO异步模型

《Netty实战》中画的已经非常清楚，这里我照搬了下来。

![](http://oq1pehpfd.bkt.clouddn.com/blog/nio%E4%BC%A0%E8%BE%93%E6%A8%A1%E5%9E%8B.png-blog)



