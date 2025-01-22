# Channel 通道的主要成员和方法
Netty 通道的抽象类 AbstractChannel 的构造函数如下：
```java
protected AbstractChannel(Channel parent) {  
    this.parent = parent;  
    id = newId();  
    unsafe = newUnsafe();  
    pipeline = newChannelPipeline();  
}
```
AbstractChannel 内部有一个 pipeline 属性，表示处理器的流水线。Netty 在对通道进行初始化的时候，将 pipeline 属性初始化为 DefaultChannelPipeline 的实例。每个通道拥有一条 ChannelPipeline 处理器流水线。
AbstractChannel 内部有一个 parent 父通道属性，保持通道的父通道。对于连接监听通道（如NioServerSocketChannel）来说，其父亲通道为 null；而对于传输通道（如 NioSocketChannel）来说，其 parent 属性的值为接收到该连接的监听通道。
几乎所有的Netty通道实现类都继承了 AbstractChannel 抽象类，都拥有上面的 parent 和pipeline 两个属性成员。

接下来，介绍一下 Channel 中的几个重要方法：
```java
	/* 连接远程服务器，方法参数为远程服务器的地址，调用后会立即返回一个执行连接操作的异步任务 ChannelFuture。 */
	ChannelFuture connect(SocketAddress remoteAddress)
	
	/* 绑定监听地址，开始监听新的客户端连接，此方法在服务器的新链接监听和接收通道的时候使用 */
	ChannelFuture bind(Socket Address address);
	
	/* 关闭通道连接，返回关闭通道连接的异步任务。 */
	ChannelFuture close();
	
	/*.读取通道数据，并且启动入站处理。
	具体来说，从内部的 Java NIO Channel 记性你数据的读取，然后启动内部的 pipeline 流水线，开始数据读取的入站处理 */
	Channel read();
	
	/* 启动出站的流程， 把处理的最终数据写入到底层的通道，返回出站处理的异步任务。 */
	ChannelFuture write(Object o)
	
	/* 将缓冲区中对数据立刻写入对端，前面的 write 方法是将数据写到操作系统的缓冲区，
	这个方法则可以将缓冲区中对数据立刻写入对端。 */
	Channel flush()
```

# EmbeddedChannel 嵌入式通道
在 Netty 中，底层传输通信的基础工作，Netty 已经完成了，主要还是开发 ChannelHandler 业务处理器。
处理器开发完成后，需要投入单元测试。一般单元测试的大致流程是：
	需要将Handler业务处理器加入到通道的Pipeline流水线中；
	接下来先后启动Netty服务器、客户端程序；
	相互发送消息，测试业务处理器的效果。

这些复杂的工序存在一个问题：如果每开发一个业务处理器，都进行服务器和客户端的重复启动，这整个的过程是非常的烦琐和浪费时间的。
如何解决这种徒劳的、低效的重复工作呢？Netty提供了一个专用通道——名字叫 EmbeddedChannel（嵌入式通道）。
EmbeddedChannel 仅仅是模拟入站与出站的操作，底层不进行实际的传输，不需要启动 Netty 服务器和客户端。除了不进行传输之外，EmbeddedChannel 的其他的事件机制和处理流程和真正的传输通道是一模一样的。因此，使用 EmbeddedChannel，开发人员可以在单元测试用例中方便、快速地进行 ChannelHandler 业务处理器的单元测试。
为了模拟数据的发送和接收，EmbeddedChannel 提供了一组专门的方法：

| 方法名称               | 说明                                                                     |
| ------------------ | ---------------------------------------------------------------------- |
| writeInbound(...)  | 向通道写入入站数据，模拟真实通道收到数据的场景。也就是说，这些写入的数据会被流水线上的入站处理器所处理到。                  |
| readInbound(...)   | 从 EmbeddedChannel 中读取入站数据，返回经过流水线最后一个入站处理器处理完成之后的入站数据。如果没有数据，则返回 null。 |
| writeOutbound(...) | 向通道写入出站数据，模拟真实通道发送数据。也就是说，这些写入的数据会被流水线上的出站处理器处理。<br>                   |
| readOutbound(...)  | 从 EmbeddedChannel 中读取出站数据，返回经过流水线最后一个出站处理器处理之后的出站数据。如果没有数据，则返回 null。   |
| finish()           | 结束 EmmbeddedChannel，会调用 Channel 的 close 方法。                            |
![[入站和出站示意图.png|1000]]
方法 1. writeInbound 入站数据写到通道
它的使用场景是：用于测试入站处理器。在测试入站处理器时（例如测试一个解码器），需要读取入站（Inbound）数据。
可以调用 writeInbound 方法，向 EmbeddedChannel 写入一个入站数据（如二进制ByteBuf数据包），模拟底层的入站包，从而被入站处理器处理到，达到测试的目的。
readInboud 和 writeInbound 方法配合使用，获取入站数据被处理后的最终形式。

方法 2. writeOutbound 出站数据写到通道
它的使用场景是：用于测试出站处理器。在测试出站处理器时（例如测试一个编码器），需要有出站的（Outbound）数据进入到流水线。可以调用writeOutbound方法，向模拟通道写入一个出站数据（如二进制ByteBuf数据包），该包将进入处理器流水线，被待测试的出站处理器所处理。
readOutbound 与这个方法配合使用，获取 writeOutbound 方法中出站数据最终被处理之后的形式。
