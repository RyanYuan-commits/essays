---
type: Java
sub-type: Framework
topic: Dubbo
---
## 1	总览

Dubbo 的 Remoting 层提供了多种客户端和服务端通信的功能, 包含了 Exchange, Transport, Serialize 三个子层级.

Dubbo 并没有自己实现一套完整的网络库, 而是使用了现有的, 相对成熟的第三方网络库, 例如 Netty, Mina 或是 Grizzly 等 NIO 框架, 可以根据实际使用场景修改配置, 选择底层使用的 NIO 框架.

dubbo-remoting 模块中的 dubbo-remoting-api 是其他模块的顶层抽象, 模块结构为:

![[dubbo-remoting-api.png]]

- buffer 包: 定义了缓冲区相关的接口, 抽象类和实现类, 缓冲区是 NIO 框架不可获取的角色, 在各个 NIO 框架中都有自己的缓冲区实现, 这里的缓冲区是更高层面的抽象, 抽象了各个 NIO 框架的缓冲区, 同时也提供了一些基础实现;
	
- exchange 包: 抽象了 Request 和 Reponse 两个概念, 并为其添加了很多特性, 是整个远程调用中非常核心的部分;

- transport 包: 对于网络传输层的抽象,但它只负责抽象单向的消息传输, 即请求从 Client 端发出, Server 端接收; 响应消息从 Server 端发出, Client 端发出, 有很多网络库可以实现网络传输的功能, 例如 Netty, Grizzly 等, transport 包是在这些网络库上层的一层抽象;
	
- 其他接口: Endpoint, Channel, Transport, Dispatcher 等顶层接口也在这个包中, 这些接口是 Dubbo Remoting 的核心接口.

## 2	传输层核心接口

### 2.1	Endpoint

在 Dubbo 中, 抽象出来了一个断点 Endpoint 的概念, 可以认为一个 ip 和一个 port 唯一确定一个端点; 

![[Dubbo Endpoint Interface.png]]

提供了 get 类方法来获取 Endpoint 本身的一些属性, 其中包括 Endpoint 本地地址, 关联 URL, 以及底层 Channel  关联的 ChannelHandler

send 方法负责数据的发送

最后几个 close 类的方法, 用于关闭底层的 Channel, isClosed 方法用于检测底层的 Channel 是否已经关闭

### 2.2	Channel

站在某一端的视角看对端可以看作是一个 Channel, 发送端通过接收端的 Channel 来发送消息, 和 Netty Channel 的概念基本一致.

![[Dubbo Channel Interface.png]]

Channel 继承了 Endpoint, 也具备开关状态和发送数据的能力, 并且还具有携带 KV 属性的能力

### 2.3	ChannelHandler

ChannelHandler 是注册在 Channel 上的消息处理器, Netty 中也有类似的抽象, 在 ChannelHandler 中可以处理 Channel 的连接建立以及连接断开事件, 还可以处理发送, 接收到的数据, 处理捕获到的异常, 从方法的命名全部是过去式可以看出来, 这些都是已经发生过的时间, ChannelHandler 是在这个事件发生过之后进行处理的.

```java
/**
 * ChannelHandler. (API, Prototype, ThreadSafe)
 *
 * @see org.apache.dubbo.remoting.Transporter#bind(org.apache.dubbo.common.URL, ChannelHandler)
 * @see org.apache.dubbo.remoting.Transporter#connect(org.apache.dubbo.common.URL, ChannelHandler)
 */
@SPI(scope = ExtensionScope.FRAMEWORK)
public interface ChannelHandler {

    /**
     * on channel connected.
     *
     * @param channel channel.
     */
    void connected(Channel channel) throws RemotingException;

    /**
     * on channel disconnected.
     *
     * @param channel channel.
     */
    void disconnected(Channel channel) throws RemotingException;

    /**
     * on message sent.
     *
     * @param channel channel.
     * @param message message.
     */
    void sent(Channel channel, Object message) throws RemotingException;

    /**
     * on message received.
     *
     * @param channel channel.
     * @param message message.
     */
    void received(Channel channel, Object message) throws RemotingException;

    /**
     * on exception caught.
     *
     * @param channel   channel.
     * @param exception exception.
     */
    void caught(Channel channel, Throwable exception) throws RemotingException;
}
```

ChannelHandler 是被 @SPI 修饰的, 是一个拓展点

### 2.4	Codec2

```java
@SPI(scope = ExtensionScope.FRAMEWORK)
public interface Codec2 {

    @Adaptive({Constants.CODEC_KEY})
    void encode(Channel channel, ChannelBuffer buffer, Object message) throws IOException;

    @Adaptive({Constants.CODEC_KEY})
    Object decode(Channel channel, ChannelBuffer buffer) throws IOException;

    enum DecodeResult {
        NEED_MORE_INPUT,
        SKIP_SOME_INPUT
    }
}
```

