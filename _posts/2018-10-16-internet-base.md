---
layout:     post
title:      "计算机网络 相关问题"
subtitle:   "TCP/IP等"
date:       2018-10-16
author:     "jiangydev"
header-img: "img/post-bg-cloud.jpg"
header-mask: 0.3
catalog:    true
tags:
    - 计算机网络
---

[TOC]

## 计算机网络 相关问题

### 1 TCP 连接断开的过程和 TIME_WAIT 存在的原因 

大多数 TCP 实现允许在连接终止时有两种选择：三向握手、具有半关闭选项的四向握手。

#### 1.1 TCP 连接断开过程

- 三向握手

![TCP连接断开-三向握手](/img/in-post/internet/TCP连接断开-三向握手.png)

- 半关闭

![TCP连接断开-半关闭](/img/in-post/internet/TCP连接断开-半关闭.png)

#### 1.2 TIME_WAIT 存在的原因

> 参考文献：
>
> [TCP连接TIME_WAIT和CLOSE_WAIT状态](https://blog.csdn.net/renwotao2009/article/details/50778399)

- 可靠地实现 TCP 全双工连接的终止

  TCP 协议在关闭连接的四次握手过程中，最终的 ACK 是由主动关闭连接的一端（后面统称A端）发出的，如果这个 ACK 丢失，被动方（后面统称B端）将重发出最终的FIN，因此A端必须维护状态信息（TIME_WAIT）允许它重发最终的ACK。如果A端不维持TIME_WAIT状态，而是处于 CLOSED  状态，那么A端将响应 RST 分节（拒绝连接），B端收到后将此分节解释成一个错误（在 Java 中会抛出 connection reset 的 SocketException)，会导致服务器无法关闭连接。因而，要实现 TCP 全双工连接的正常终止，必须处理终止过程中四个分节任何一个分节的丢失情况，主动关闭连接的A端必须维持TIME_WAIT状态 。

- 允许老的重复 segment 在网络中消逝 

  某个连接中重复报文段可能出现在下一个连接中。假定客户和服务器已经关闭连接，经过短暂的时间后，他们又打开一个新的连接，使用的相同的套接字地址（源地址，目的地址，源端口号，目的端口号相同），这样的新连接称为旧连接的化身，那么前一个连接中的重复报文有可能会达到新连接中。为避免这个问题，TCP 规定必须经过 2MSL 时间。

#### 1.3 TIME_WAIT 维持的时间

MSL就是 Maximum Segment Lifetime(最大分节生命期），这是一个 IP 数据包能在互联网上生存的最长时间，超过这个时间 IP 数据包将在网络中消失 。MSL 在 RFC 1122上建议是 2 分钟，而源自 Berkeley 的 TCP 实现传统上使用 30 秒。

- TIME_WAIT 维持时间较短

1. 旧连接中的重复报文可能会被新连接接收。
2. ACK 报文丢失，主动方如果想建立新的连接（发送 SYN 报文），而被动方会返回 RST，导致无法建立新的连接。

- TIME_WAIT 维持时间较长

  主动方无法关闭连接，占用程序资源。