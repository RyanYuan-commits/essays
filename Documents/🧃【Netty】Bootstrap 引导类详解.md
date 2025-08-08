## 1 基础概念

在 Netty 定义了两种引导类, 分别在服务器端和客户端使用: 

![[Netty 中的两个引导类.png|500]]

它们大致的配置和使用方法都是相同的. 下面以 `ServerBootstrap` 为例介绍.

## 2 父子通道

在 Netty 中, 每一个 `NioSocketChannel` 通道所封装的是 Java NIO 通道, 再往下就对应到了操作系统底层的 `socket` 文件描述符.

理论上来说, 操作系统底层的 socket 文件描述符分为两类:

- **连接监听类型**. 连接监听类型的 socket 描述符, 处于服务器端, 它负责接收客户端的套接字连接; 在服务器端, 一个“连接监听类型”的 socket 描述符可以接受成千上万的传输类的 socket 文件描述符.
- **数据传输类型**. 数据传输类的 socket 描述符负责传输数据. 同一条 TCP 的 Socket 传输链路, 在服务器和客户端, 都分别会有一个与之相对应的数据传输类型的 socket 文件描述符.

在 Netty 中, 异步非阻塞的服务器端监听通道 `NioServerSocketChannel`, 所封装的 Linux 底层的文件描述符, 是“连接监听类型”的 socket 描述符;

而异步非阻塞的传输通道 `NioSocketChannel`, 所封装的 Linux 的文件描述符, 是“数据传输类型”的 socket 描述符.

在 Netty 中, 将有**接收关系**的监听通道和传输通道, 叫做父子通道.

负责服务器连接监听和接收的监听通道 (如 `NioServerSocketChannel`), 称为父通道.
对应于每一个接收到的传输类通道 (如 `NioSocketChannel`), 称为子通道.

## 3 EventLoopGroup 事件轮询组

实际上在 Netty 中, 一个 `EventLoop` 相当于一个子反应器 (`SubReactor`), 一个 `NioEventLoop` 子反应器拥有了一个事件轮询 `Thread`, 同时拥有一个 Java NIO 选择器.

- Netty 使用 `EventLoopGroup` 轮询组来实现了多线程版本的 Reactor 模型, 多个 EventLoop 线程放在一起, 可以组成一个 EventLoopGroup 轮询组.
- 反过来说, EventLoopGroup 轮询组就是一个多线程版本的反应器, 其中的单个 EventLoop 线程对应于一个子反应器 (SubReactor) .

为了提升性能, Netty 没有直接使用单个 `EventLoop` 事件轮询器, 而是使用多个事件轮询器构成的事件轮询器组 `EventLoopGroup`;

`EventLoopGroup` 的构造函数有一个指定内部的线程数的参数. 在构造器初始化时, 会按照传入的线程数量, 在内部构造多个 `Thread` 和 `EventLoop` 子反应器 (一个线程对应一个 EventLoop 子反应器) , 进行多线程的 IO 事件查询和分发.

线程数的默认值为最大可用的 CPU 处理器数量的 2 倍.

![[Netty 中的 Reactor 模式示意图.png|600]]

为了及时接受到新连接, 在服务器端, 一般有两个独立的反应器, 一个反应器负责新连接的监听和接受, 另一个反应器负责 IO 事件轮询和分发, 两个反应器相互隔离.

对应到Netty 服务器程序中, 则需要设置两个 `EventLoopGroup` 轮询组, 一个组负责新连接的监听和接受, 另外一个组负责 IO 传输事件的轮询与分发, 两个轮询组的职责具体如下: 

- 负责新连接的监听和接收的 `EventLoopGroup` 轮询组中的反应器, 完成查询通道的新连接 IO 事件查询, 被称为 Boss 轮询组.
- 另一个轮询组中的反应器, 完成查询所有子通道的 IO 事件, 并且执行对应的 `Handler` 处理器完成 IO 处理, 例如数据的输入和输出, 被称为 Worker 轮训组.

## 4 BootStrap 详解

BootStrap 的启动流程, 也就是 Netty 组件的组装, 配置, 以及 Netty 服务器或者客户端的启动流程, 首先创建一个服务器端的引导实例: 

```java
ServerBootstrap bootstrap = new ServerBootstrap();
```

### 4.1 创建 Reactor 轮询组

```java
NioEventLoopGroup bossLoopGroup = new NioEventLoopGroup();  
NioEventLoopGroup workerLoopGroup = new NioEventLoopGroup();  
// 设置反应器轮训组  
bootstrap.group(bossLoopGroup, workerLoopGroup);
```

在设置之前, 创建了两个 `NioEventLoopGroup` 轮询组, 分别负责连接事件的处理和 IO 事件的处理.

如果不需要进行新连接事件和输出事件进行分开监听, 就不一定非得配置两个轮询组, 可以仅配置一个 `EventLoopGroup` 反应器轮询组.

但是, 在这种模式下, 新连接监听 IO 事件和数据传输 IO 事件可能被挤在了同一个线程中处理.这样会带来一个风险: 新连接的接受被更加耗时的数据传输或者业务处理所阻塞. 

所以, 在服务器端, 建议设置成两个轮询组的工作模式.

### 4.2 设置通道的 IO 类型

Netty 不止支持 Java NIO, 也支持阻塞式的 OIO, 可以通过切换不同的 Channel 来切换 IO 类型: 

```java
// step2: 设置传输通道的类型为 nio 类型
b.channel(NioServerSocketChannel.class);
```

如果要指定 OIO 类型, 配置为 `OioServerSocketChannel.class` 类即可. 但由于NIO的优势巨大, 通常不会在 Netty 中使用 BIO.

### 4.3 设置监听端口

```java
// step3: 设置监听端口
b.localAddress(new InetSocketAddress(port));
```

### 4.4 设置传输通道配置选项

```java
// step4: 设置通道的参数
b.option(ChannelOption.SO_KEEPALIVE, true); // 开启 TCP 底层心跳机制
b.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
```

这里用到了 `Bootstrap` 的 `option` 选项设置方法.

上面提到, 在服务端有父子通道的概念, 也自然会有父子通道的设置, `option` 方法用于给父通道设置一些与传输协议相关的选项.如果要给子通道设置一些通道选项, 则需要使用 `childOption`.

### 4.5 装配子通道的 Pipeline 流水线

每一个通道都拥有一条 `ChannelPipeline`, 通过双向链表来维护与 `Channel` 绑定的 `Handler`, 

通过重写 `ChannelInitializer` 的 `initChannel` 方法, 来指定需要添加的 `Handler`.

```java
// 装配子通道流水线  
bootstrap.childHandler(new ChannelInitializer<SocketChannel>() {  
    // 有连接到达的时候会创建一个通道  
    @Override  
    protected void initChannel(SocketChannel ch) throws Exception {  
        // 流水线: 负责管理通道中的 Handler 处理器  
        // 向子通道流水线添加一个 Handler 处理器  
        ch.pipeline().addLast(new NettyDiscardHandler());  
    }  
});
```

父通道负责处理连接事件, 一般不需要配置与业务逻辑相关的 Handler, 如果需要完成一些特殊处理, 可以通过 `handler(ChannelHandler)` 方法来配置.

### 4.6 绑定服务器的监听端口

```java
// step6: 开始绑定端口, 通过调用 sync 同步方法阻塞直到绑定成功
ChannelFuture channelFuture = b.bind().sync();
Logger.info(" 服务器启动成功, 监听端口: " + channelFuture.channel().localAddress());
```

`b.bind()` 方法会返回一个端口绑定 Netty 的异步任务 `ChannelFuture`. 通过 `sync` 方法, 同步阻塞等待端口绑定的完成.

返回的是一个 `ChannelFuture` 异步实例, 实际上, Netty 中的 IO 操作, 都会返回异步任务实例.

可以选择同步阻塞一直到异步任务执行完成, 也可以注册异步回调逻辑.

### 4.7 自我阻塞直到监听通道关闭

```java
// step7: 自我阻塞, 直到通道关闭的异步任务结束
ChannelFuture closeFuture = channelFuture.channel().closeFuture();
closeFuture.sync();
```

如果要阻塞当前线程直到通道关闭, 可以使用通道的 `closeFuture()` 方法, 以获取通道关闭的异步任务.当通道被关闭时, `closeFuture` 实例的 `sync()` 方法会返回.

### 4.8 关闭 EventLoopGroup

```java
// step8: 释放掉所有资源, 包括创建的反应器线程
workerLoopGroup.shutdownGracefully();
bossLoopGroup.shutdownGracefully();
```

关闭事件轮询组, 同时会关闭内部的 `SubReactor` 子反应器线程, 也会关闭内部的 `Selector` 选择器, 内部的轮询线程以及负责查询的所有的子通道.

在子通道关闭后, 会释放掉底层的资源, 如 Socket 文件描述符等.

## 5 Channel Option 整理

| 选项名称                  | 类型         | 描述                                                    | 默认值                           | 关键点                                                 |
| --------------------- | ---------- | ----------------------------------------------------- | ----------------------------- | --------------------------------------------------- |
| SO_RCVBUF / SO_SNDBUF | 接收/发送缓冲区   | 设置 TCP 连接的内核接收与发送缓冲区大小，与 TCP 全双工和滑动窗口机制有关。            | 无固定默认值，由操作系统决定。               | 影响吞吐量，缓冲区越大，可容纳更多数据，但可能增加内存占用。                      |
| TCP_NODELAY           | 延时发送       | 控制是否启用 Nagle 算法。true 立即发送小数据包（禁用），false 累积数据后再发送（启用）。 | Netty 默认为 true，操作系统默认为 false。 | 影响实时性，禁用时减少延迟，启用时减少网络交互次数。                          |
| SO_KEEPALIVE          | 心跳机制       | 开启 TCP 协议的心跳机制，用于探测空闲连接的有效性。                          | FALSE                         | 防止空闲连接被断开。默认心跳间隔为 2 小时，Netty 默认关闭。                  |
| SO_REUSEADDR          | 地址复用       | 允许在特定情况下重用地址和端口，特别是当一个连接处于 TIME_WAIT 状态时。             | FALSE                         | 解决端口占用问题，方便服务快速重启。                                  |
| SO_LINGER             | close 行为控制 | 控制 socket.close() 方法的行为。可设置立即返回、丢弃缓冲区数据或阻塞至数据发送完毕。    | -1                            | 影响连接关闭行为。-1 立即返回并发送数据，0 立即返回并丢弃数据，>0 阻塞直到数据发送完毕或超时。 |
| SO_BACKLOG            | TCP 队列大小   | 定义服务器端接收连接的队列长度。队列满时，新连接将被拒绝。                         | 无固定默认值，由操作系统决定。               | 影响连接处理能力，队列过小可能导致新客户端无法连接。                          |
| SO_BROADCAST          | 广播模式       | 允许将数据包设置为广播模式发送。                                      | FALSE                         | 用于 UDP 多播等特殊场景，不用于 TCP。                             |
