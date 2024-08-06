---
title: Netty的架构
tags:
  - netty
  - netty基础知识
status: Done
createDate: 2023-12-11
---

## 架构设计

![[架构设计.png]]

- core，核心层，主要定义一些基础设施，比如事件模型、通信 API、缓冲区等。
- Transport Service，传输服务层，主要定义一些通信的底层能力，或者说是传输协议的支持，比如 TCP、UDP、HTTP 隧道、虚拟机管道等。
- Protocol Support，协议支持层，这里的协议比较广泛，不仅仅指编解码协议，还可以是应用层协议的编解码，比如 HTTP、WebSocket、SSL、Protobuf、文本协议、二进制协议、压缩协议、大文件传输等，基本上主流的协议都支持。

## netty-common

- 通用的工具类，比如 `StringUtil` 等
- 对于 JDK 原生类的增强，比如 `Future` 、`FastThreadLocal` 等
- Netty 自定义的并发包，比如 `EventExecutor` 等
- Netty 自定义的集合包，主要是对 HashMap 的增强

## netty-buffer

Netty 自己实现的 Buffer，做了很多优化，比如池化 Buffer、组合 Buffer 等都是在这个包里，Netty 把性能优化到了极致。

## netty-resolver

主要是做地址解析用的。

## netty-transport

`netty-transport` 主要定义了一些服务于传输层的接口和类，比如 Channel、ChannelHandler、ChannelHandlerContext、EventLoop等，这些接口和类都非常的酷。

> TCP，传输控制协议，Java 中一般用 SocketXxx、ServerSocketXxx 表示基于 TCP 协议通信。
> 
> UDP，用户数据报文协议，Java 中一般用 DatagramXxx 表示基于 UDP 协议通信，Datagram，数据报文的意思。
> 
> SCTP，流控制传输协议。
> 
> RXTX，串口通讯协议。
> 
> UDT，基于 UDP 的数据传输协议。

## netty-handler

`netty-handler` 中定义了各种不同的 Handler，满足不同的业务需求，这些 Handler 都是 Netty 中非常棒的功能，比如，**IP 过滤、日志、SSL、空闲检测、流量整形等**，有了这些 Handler，我们不仅能让我们的程序可运行，更能使我们的程序安全地运行，非常棒。

## netty-codec

`netty-codec` 中定义了一系列编码解码器，比如，base64、json、marshalling、protobuf、serializaion、string、xml 等，几乎市面上能想到的编码、解码、序列化、反序列化方式，Netty 中都可以支持，它们是一类特殊的 ChannelHandler，专门负责编解码的工作。