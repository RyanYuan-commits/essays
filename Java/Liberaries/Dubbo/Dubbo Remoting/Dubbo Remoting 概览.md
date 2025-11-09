---
type: Java
sub-type: Framework
topic: Dubbo
---
## 1	总览

Dubbo 的 Remoting 层提供了多种客户端和服务端通信的功能, 包含了 Exchange, Transport, Serialize 三个子层级.

Dubbo 并没有自己实现一套完整的网络库, 而是使用了现有的, 相对成熟的第三方网络库, 例如 Netty, Mina 或是 Grizzly 等 NIO 框架, 可以根据实际使用场景修改配置, 选择底层使用的 NIO 框架.

dubbo-remoting 模块中的 dubbo-remoting-api 是其他模块的顶层抽象, 有这些关键包:

- buffer 包: 定义了缓冲区相关的接口, 抽象类和实现类, 缓冲区是 NIO 框架不可获取的角色, 在各个 NIO 框架中都有自己的缓冲区实现, 这里的缓冲区是更高层面的抽象, 抽象了各个 NIO 框架的缓冲区, 同时也提供了一些基础实现;
	
- exchange 包: 抽象了 Request 和 Reponse 两个概念, 并为其添加了很多特性, 是整个远程调用中非常核心的部分;

- transport 包: 对于网络传输层的抽象,但它只负责抽象单向的消息传输, 即请求从 Client 端发出, Server 端接收; 响应消息从 Server 端发出, Client 端发出, 有很多网络库可以实现网络传输的功能, 例如 Netty, Grizzly 等, transport 包是在这些网络库上层的一层抽象;
	
- 其他接口: Endpoint, Channel, Transport, Dispatcher 等顶层接口也在这个包中, 这些接口是 Dubbo Remoting 的核心接口.

## 2	传输层核心接口

### 2.1	Endpoint

Dubbo 中抽象出来了一个端点 Endpoint 的概念, 可以认为一个 ip 和一个 port 唯一确定一个端点; 

![[Dubbo Endpoint Interface.png]]

提供了 get 类方法来获取 Endpoint 本身的一些属性, 其中包括 Endpoint 本地地址, 关联 URL, 以及底层 Channel  关联的 ChannelHandler;

send 方法负责数据的发送;

最后几个 close 类的方法, 用于关闭底层的 Channel, isClosed 方法用于检测底层的 Channel 是否已经关闭.

### 2.2	Channel

站在某一端的视角看对端可以看作是一个 Channel, 代表一次激活的通信, 和 Netty Channel 的概念基本一致.

![[Dubbo Channel Interface.png]]

Channel 继承了 Endpoint, 也具备开关状态和发送数据的能力, 并且还具有携带 KV 属性的能力

### 2.3	ChannelHandler

ChannelHandler 是注册在 Channel 上的消息处理器,在 ChannelHandler 中可以处理 Channel 的连接建立以及连接断开事件, 还可以处理发送, 接收到的数据, 处理捕获到的异常;

方法的命名全部是过去式, 这些都是已经发生过的事件, ChannelHandler 是在这个事件发生过之后进行处理的.

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

ChannelHandler 是被 @SPI 修饰的, 是一个拓展点.

### 2.4	Codec2

```java
@SPI(scope = ExtensionScope.FRAMEWORK)
public interface Codec2 {

    @Adaptive({Constants.CODEC_KEY})
    void encode(Channel channel, ChannelBuffer buffer, Object message) throws IOException;

    @Adaptive({Constants.CODEC_KEY})
    Object decode(Channel channel, ChannelBuffer buffer) throws IOException;

    enum DecodeResult {
    
        NEED_MORE_INPUT, SKIP_SOME_INPUT
        
    }
    
}
```

Netty 中, 有一类特殊的 ChannelHandler 专门负责实现编解码功能, 从而实现字节数据与有意义消息之间, 或者消息之间的互相转化, 在 Dubbo Remoting 中也有类似的抽象. 上面的 Codec2 接口被 @SPI 修饰, 表示这个接口是一个拓展接口, 同时 encode 方法和 decode 方法都被 @Adaptive 修饰, 可以通过 URL 来指定具体的拓展实现类.

DecodeResult Enum 是在处理 TCP 粘包和拆包时使用的, 当读取到的数据不足以构成一个消息的时候, 就会使用 NEED_MORE_INPUT 这个枚举.

### 2.5	Client & RemotingServer

两个接口分别抽象了客户端和服务端, 两者都继承了 Channel, Resetable 等接口, 也就是说, 两者都具备读写数据的能力.

![[Dubbo CLient and RemotingServer Interface.png]]

Client 和 RemotingServer 的主要区别是, Client 只能关联一个 Channel, 而 RemotingServer 可以接收 Client 发起的多个 Channel, 所以在 RemotingServer 接口中定义了查询 Channel 的相关方法

```java
/**
 * Remoting Server. (API/SPI, Prototype, ThreadSafe)
 * <p>
 * <a href="http://en.wikipedia.org/wiki/Client%E2%80%93server_model">Client/Server</a>
 *
 * @see org.apache.dubbo.remoting.Transporter#bind(org.apache.dubbo.common.URL, ChannelHandler)
 */
public interface RemotingServer extends Endpoint, Resetable, IdleSensible {

    /**
     * is bound.
     *
     * @return bound
     */
    boolean isBound();

    /**
     * get channels.
     *
     * @return channels
     */
    Collection<Channel> getChannels();

    /**
     * get channel.
     *
     * @param remoteAddress
     * @return channel
     */
    Channel getChannel(InetSocketAddress remoteAddress);

    @Deprecated
    void reset(org.apache.dubbo.common.Parameters parameters);
    
}
```

### 2.6	Transporter

Dubbo 在 Client 和 RemotingServer 之上有封装了一层工厂接口, 用来获取 Client 和 RemotingServer 实例:

```java
/**
 * Transporter. (SPI, Singleton, ThreadSafe)
 * <p>
 * <a href="http://en.wikipedia.org/wiki/Transport_Layer">Transport Layer</a>
 * <a href="http://en.wikipedia.org/wiki/Client%E2%80%93server_model">Client/Server</a>
 *
 * @see org.apache.dubbo.remoting.Transporters
 */
@SPI(value = "netty", scope = ExtensionScope.FRAMEWORK)
public interface Transporter {

    /**
     * Bind a server.
     *
     * @param url     server url
     * @param handler
     * @return server
     * @throws RemotingException
     * @see org.apache.dubbo.remoting.Transporters#bind(URL, ChannelHandler...)
     */
    @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})
    RemotingServer bind(URL url, ChannelHandler handler) throws RemotingException;

    /**
     * Connect to a server.
     *
     * @param url     server url
     * @param handler
     * @return client
     * @throws RemotingException
     * @see org.apache.dubbo.remoting.Transporters#connect(URL, ChannelHandler...)
     */
    @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
    Client connect(URL url, ChannelHandler handler) throws RemotingException;
    
}
```

Transporter 接口上有 @SPI 注解, 是一个拓展接口, 默认使用 "netty" 这个拓展名, 并且方法都被 @Adaptive 朱姐休息, 会根据 URL 中的值确定拓展实现类.

对于每一个支持的 NIO 库, 都有一个 Transporter  接口的实现类, 比如 NettyTransporter, MinaTransporter, GrizzlyTransporter 等.

接口中返回的 Client 和 RemotingServer 是 NIO 库对应的实现类, 中间使用抽象类提供了部分默认实现.

![[Dubbo Transporter Interface Implementation.png]]

Dubbo 还提供了一个门面类, Transports, 封装了 Transporter 对象的创建以及 ChannelHandler 的处理:

```java
/**
 * Transporter facade. (API, Static, ThreadSafe)
 */
public class Transporters {

    private Transporters() {}

    public static RemotingServer bind(String url, ChannelHandler... handler) throws RemotingException {
        return bind(URL.valueOf(url), handler);
    }

    public static RemotingServer bind(URL url, ChannelHandler... handlers) throws RemotingException {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handlers == null || handlers.length == 0) {
            throw new IllegalArgumentException("handlers == null");
        }
        ChannelHandler handler;
        if (handlers.length == 1) {
            handler = handlers[0];
        } else {
            handler = new ChannelHandlerDispatcher(handlers);
        }
        return getTransporter(url).bind(url, handler);
    }

    public static Client connect(String url, ChannelHandler... handler) throws RemotingException {
        return connect(URL.valueOf(url), handler);
    }

    public static Client connect(URL url, ChannelHandler... handlers) throws RemotingException {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        ChannelHandler handler;
        if (handlers == null || handlers.length == 0) {
            handler = new ChannelHandlerAdapter();
        } else if (handlers.length == 1) {
            handler = handlers[0];
        } else {
            handler = new ChannelHandlerDispatcher(handlers);
        }
        return getTransporter(url).connect(url, handler);
    }

    public static Transporter getTransporter(URL url) {
        return url.getOrDefaultFrameworkModel()
                .getExtensionLoader(Transporter.class)
                .getAdaptiveExtension();
    }
}
```

在创建 Client 和 RemotingServer 的时候, 可以指定多个 ChannelHandler 绑定到 Channel 来处理其中传输的数据, Transporters 会将其封装成一个 ChannelHandlerDispatcher 对象.

ChannelHandlerDispatcher 也是 ChannelHandler 接口的实现类之一, 维护了一个 CopyOnWriteArraySet 集合, 它所有的 ChannelHandler 接口实现都会调用其中每个 ChannelHandler 元素的相应方法. 另外, ChannelHandlerDispatcher 还提供了增删该 ChannelHandler 集合的相关方法.

## 3	总结

- Endpoint 接口抽象了 "端点" 的概念, 这是所有抽象接口的基础.
	
- 上层使用方会通过 Transporters 门面类获取到 Transporter 的具体扩展实现, 然后通过 Transporter 拿到相应的 Client 和 RemotingServer 实现, 就可以建立（或接收）Channel 与远端进行交互了.
	
- 无论是 Client 还是 RemotingServer, 都会使用 ChannelHandler 处理 Channel 中传输的数据, 其中负责编解码的 ChannelHandler 被抽象出为 Codec2 接口.

![[Dubbo Transporter 层整体架构.png|800]]