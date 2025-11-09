---
type: Java
sub-type: Framework
topic: Dubbo
---
> 官方介绍: 抽象 mina 和 netty 为统一接口, 以 Message 为中心, 扩展接口为 Channel, Transporter, Client, Server, Codec.

![[Dubbo Transport.png]]

## 1	AbstractPeer

AbstractPeer 抽象类同时实现了 [[Dubbo Remoting 概览#2.1 Endpoint|Endpoint]] 和 [[Dubbo Remoting 概览#2.3 ChannelHandler|ChannelHandler]] 接口, 代表在网络通信中一个对等的, 可交互的端点, Dubbo 