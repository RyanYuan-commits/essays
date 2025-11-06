---
type: Java
sub-type: Framework
topic: Dubbo
---
Dubbo 的 Remoting 层提供了多种客户端和服务端通信的功能, 包含了 Exchange, Transport, Serialize 三个子层级.

Dubbo 并没有自己实现一套完整的网络库, 而是使用了现有的, 相对成熟的第三方网络库, 例如 Netty, Mina 或是 Grizzly 等 NIO 框架, 可以根据实际使用场景修改配置, 选择底层使用的 NIO 框架.

dubbo-remoting 模块中的 dubbo-remoting-api 是其他模块的顶层抽象, 模块结构为:

![[dubbo-remoting-api.png]]

- buffer 包: 定义了缓冲区相关的接口, 抽象类和实现类, 缓冲区是 NIO 框架不可获取的角色, 在各个 NIO 框架中都有自己的缓冲区实现, 这里的缓冲区是更高层面的抽象, 抽象了各个 NIO 框架的缓冲区, 同时也提供了一些基础实现;
	
- exchange 包: 抽象了 Request 和 Reponse 两个概念, 并为其添加了很多特性, 是整个远程调用中非常核心的部分;

- transport 包:  