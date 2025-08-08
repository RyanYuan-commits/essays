## 1 Channel 通道的主要成员和方法

Netty 通道的抽象类 AbstractChannel 的构造函数如下: 

```java
protected AbstractChannel(Channel parent) {  
    this.parent = parent;  
    id = newId();  
    unsafe = newUnsafe();  
    pipeline = newChannelPipeline();  
}
```

### 1.1 核心属性

`pipeline` 流水线属性. Netty 在对通道进行初始化的时候, 将 pipeline 属性初始化为 `DefaultChannelPipeline` 的实例. 每个通道拥有一条 `ChannelPipeline` 处理器流水线. 

`parent` 父通道属性. 连接监听通道 (如 `NioServerSocketChannel`) 的 parent 为 `null`；传输通道 (如 `NioSocketChannel`), parent 的值为接收到该连接的监听通道. 

### 1.2 重要方法

接下来, 介绍一下 Channel 中的几个重要方法: 

连接远程服务器, 方法参数为远程服务器的地址, 调用后会立即返回一个执行连接操作的异步任务 ChannelFuture.

```java
ChannelFuture connect(SocketAddress remoteAddress)
```

绑定监听地址, 开始监听新的客户端连接, 此方法在服务器的新链接监听和接收通道的时候使用.

```java
ChannelFuture bind(Socket Address address);
```

关闭通道连接, 返回关闭通道连接的异步任务.

```java
ChannelFuture close();
```

读取通道数据, 并且启动入站处理.  具体来说, 从内部的 Java NIO Channel 进行数据的读取, 然后启动内部的 pipeline 流水线, 开始数据读取的入站处理.

```java
Channel read();
```

向通道写入数据, 启动出站的流程, 把处理的最终数据写入到底层的通道, 返回出站处理的异步任务.

```java
ChannelFuture write(Object o)
```

将缓冲区中对数据立刻写入对端, 前面的 `write` 方法是将数据写到操作系统的缓冲区, 这个方法则可以将缓冲区中对数据立刻写入对端.

```java
Channel flush()
```

## 2 EmbeddedChannel 嵌入式通道

### 2.1 什么是 EmbeddedChannel?

Netty 已经封装好了底层传输通信的基础工作, 开发者主要关注的还是业务 `Handler` 的开发. 

`Hander` 开发完成后, 如果投入测试, 需要同时启动服务器和客户端; 也就是每次测试都需要启动两个服务, 做很多额外的操作, 但我们真正关注的是 `Channel` 中的 `Handler`.

为了解决这个问题, Netty 提供了一个专用通道——名字叫 `EmbeddedChannel` (嵌入式通道). 

`EmbeddedChannel` 可以很方便的模拟入站与出站的操作, 底层不进行实际的传输, 不需要启动 Netty 服务器和客户端. 除了不进行传输之外, `EmbeddedChannel` 的其他的事件机制和处理流程和真正的传输通道是一模一样的. 

因此, 使用 `EmbeddedChannel`, 开发人员可以方便、快速的进行单元测试; 为了模拟数据的发送和接收, EmbeddedChannel 提供了一组专门的方法: 

| 方法名称               | 说明                                                                     |
| ------------------ | ---------------------------------------------------------------------- |
| writeInbound(...)  | 向通道写入入站数据, 模拟真实通道收到数据的场景. 也就是说, 这些写入的数据会被流水线上的入站处理器所处理到.                   |
| readInbound(...)   | 从 EmbeddedChannel 中读取入站数据, 返回经过流水线最后一个入站处理器处理完成之后的入站数据. 如果没有数据, 则返回 null.  |
| writeOutbound(...) | 向通道写入出站数据, 模拟真实通道发送数据. 也就是说, 这些写入的数据会被流水线上的出站处理器处理. <br>                   |
| readOutbound(...)  | 从 EmbeddedChannel 中读取出站数据, 返回经过流水线最后一个出站处理器处理之后的出站数据. 如果没有数据, 则返回 null.    |
| finish()           | 结束 EmmbeddedChannel, 会调用 Channel 的 close 方法.                             |
![[入站和出站示意图.png|600]]

### 2.2 主要方法

#### writeInbound 入站数据写到通道

它的使用场景是: 用于测试入站处理器. 在测试入站处理器时 (如解码器), 需要读取入站数据. 

可以调用 `writeInbound` 方法, 向 `EmbeddedChannel` 写入一个入站数据（如二进制ByteBuf数据包）, 模拟底层的入站包, 从而被入站处理器处理到, 达到测试的目的. `readInboud` 和 `writeInbound` 方法配合使用, 获取入站数据被处理后的最终形式. 

#### writeOutbound 出站数据写到通道

它的使用场景是: 用于测试出站处理器. 在测试出站处理器时 (例如测试一个编码器), 需要有出站的 数据进入到流水线. 

可以调用 `writeOutbound` 方法, 向模拟通道写入一个出站数据（如二进制 `ByteBuf` 数据包）, 该包将进入处理器流水线, 被待测试的出站处理器所处理.`readOutbound` 与这个方法配合使用, 获取 `writeOutbound` 方法中出站数据最终被处理之后的形式. 