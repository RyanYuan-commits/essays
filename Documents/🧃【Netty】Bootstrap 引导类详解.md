# 基础概念
在 Netty 中，有两个引导类，分别用在服务器端和客户端：
![[Netty 中的两个引导类.png]]
这两个引导类仅是使用的地方不同，它们大致的配置和使用方法都是相同的。下面以 ServerBootstrap 服务器引导类作为重点的介绍对象。
## 父子通道
在Netty中，每一个 NioSocketChannel 通道所封装的是 Java NIO 通道，再往下就对应到了操作系统底层的 socket 文件描述符。
理论上来说，操作系统底层的socket文件描述符分为两类：
	连接监听类型。连接监听类型的 socket 描述符，处于服务器端，它负责接收客户端的套接字连接；在服务器端，一个“连接监听类型”的 socket 描述符可以接受（Accept）成千上万的传输类的 socket 文件描述符。
	数据传输类型。数据传输类的socket描述符负责传输数据。同一条 TCP 的 Socket 传输链路，在服务器和客户端，都分别会有一个与之相对应的数据传输类型的 socket 文件描述符。
在Netty中，异步非阻塞的服务器端监听通道 NioServerSocketChannel，所封装的 Linux 底层的文件描述符，是“连接监听类型”的 socket 描述符；
而异步非阻塞的传输通道 NioSocketChannel，所封装的 Linux 的文件描述符，是“数据传输类型”的socket描述符。
在 Netty 中，将**有接收关系的 监听通道 和 传输通道**，叫做父子通道。
	负责服务器连接监听和接收的监听通道（如 NioServerSocketChannel），也叫父通道（Parent Channel）。
	对应于每一个接收到的传输类通道（如 NioSocketChannel），也叫子通道（Child Channel）。
## EventLoopGroup 事件轮询组
实际上在 Netty 中，一个 EventLoop 相当于一个子反应器（SubReactor），一个 NioEventLoop 子反应器拥有了一个事件轮询 Thread，同时拥有一个 Java NIO 选择器。
	Netty 使用EventLoopGroup轮询组来实现了多线程版本的 Reactor 模型，多个 EventLoop 线程放在一起，可以组成一个 EventLoopGroup 轮询组。
	反过来说，EventLoopGroup 轮询组就是一个多线程版本的反应器，其中的单个 EventLoop 线程对应于一个子反应器（SubReactor）。

Netty的程序开发不会直接使用单个 EventLoop 事件轮询器，而是使用 EventLoopGroup 轮询组。EventLoopGroup 的构造函数有一个参数，用于指定内部的线程数。在构造器初始化时，会按照传入的线程数量，在内部构造多个 Thread 线程和多个EventLoop 子反应器（一个线程对应一个 EventLoop 子反应器），进行多线程的 IO 事件查询和分发。如果使用 EventLoopGroup 的无参数的构造函数，没有传入线程数量或者传入的数量为 0，默认的 EventLoopGroup 内部线程数量为最大可用的 CPU 处理器数量的 2 倍。假设电脑使用的是 4 核的CPU，那么在内部会启动 8 个 EventLoop 线程，相当8个子反应器（SubReactor）实例。

从前文可知，为了及时接受（Accept）到新连接，在服务器端，一般有两个独立的反应器，一个反应器负责新连接的监听和接受，另一个反应器负责 IO 事件轮询和分发，两个反应器相互隔离。对应到Netty服务器程序中，则需要设置两个 EventLoopGroup轮询组，一个组负责新连接的监听和接受，另外一个组负责 IO 传输事件的轮询与分发，两个轮询组的职责具体如下：
	（1）负责新连接的监听和接收的EventLoopGroup轮询组中的反应器（Reactor），完成查询通道的新连接IO事件查询，这些反应器有点像负责招工的包工头，因此，该轮询组可以形象地称为“包工头”（Boss）轮询组。
	（2）另一个轮询组中的反应器（Reactor），完成查询所有子通道的 IO 事件，并且执行对应的 Handler 处理器完成 IO 处理——例如数据的输入和输出（有点儿像搬砖），这个轮询组可以形象地称为“工人”（Worker）轮询组。
![[Netty 中的 Reactor 模式示意图.png|700]]
# BootStrap 详解
## Netty Server 启动的八个步骤
BootStrap 的启动流程，也就是 Netty 组件的组装、配置，以及 Netty 服务器或者客户端的启动流程。
首先创建一个服务器端的引导实例：
```java
ServerBootstrap bootstrap = new ServerBootstrap();
```

### 创建 Reactor 轮询组
大致流程如下：
```java
NioEventLoopGroup bossLoopGroup = new NioEventLoopGroup();  
NioEventLoopGroup workerLoopGroup = new NioEventLoopGroup();  
// 设置反应器轮训组  
bootstrap.group(bossLoopGroup, workerLoopGroup);
```
在设置之前，创建了两个 NioEventLoopGroup 轮询组，分别负责连接事件的处理和 IO 事件的处理。
如果不需要进行新连接事件和输出事件进行分开监听，就不一定非得配置两个轮询组，可以仅配置一个 EventLoopGroup 反应器轮询组。
在这种模式下，新连接监听 IO 事件和数据传输 IO 事件可能被挤在了同一个线程中处理。这样会带来一个风险：新连接的接受被更加耗时的数据传输或者业务处理所阻塞。所以，在服务器端，建议设置成两个轮询组的工作模式。

### 设置通道的 IO 类型
Netty 不止支持 Java NIO，也支持阻塞式的 OIO（也叫BIO，Block-IO，即阻塞式IO）。下面配置的是Java NIO类型的通道类型，大致如下：
```java
// step2：设置传输通道的类型为 nio 类型
b.channel(NioServerSocketChannel.class);
```
如果确实指定 Bootstrap 的 IO 型为BIO类型，配置为 OioServerSocketChannel.class 类即可。由于NIO的优势巨大，通常不会在Netty中使用BIO。

### 设置监听端口
```java
// step3：设置监听端口
b.localAddress(new InetSocketAddress(port));
```

### 设置传输通道配置选项
```java
// step4：设置通道的参数
b.option(ChannelOption.SO_KEEPALIVE, true);
b.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
```
这里用到了 Bootstrap 的 option(...) 选项设置方法。
对于服务器的 Bootstrap 而言，这个方法的作用是：给父通道（Parent Channel）通道设置一些与传输协议相关的选项。如果要给子通道（Child Channel）设置一些通道选项，则需要用另外一个 childOption(...)设置方法。
可以设置哪些通道选项（ChannelOption）呢？在上面的代码中，设置了一个底层 TCP 相关的选项 ChannelOption.SO_KEEPALIVE。该选项表示：是否开启 TCP 底层心跳机制，true 为开启，false 为关闭。其他的通道设置选项，参见下一小节。

### 装配子通道的 Pipeline 流水线
每一个通道都用一条 ChannelPipeline 流水线。它的内部有一个双向的链表。装配流水线的方式是：将业务处理器 ChannelHandler 实例包装之后加入双向链表中。
如何装配 Pipeline 流水线呢？装配子通道的 Handler 流水线调用引导类的 childHandler() 方法，该方法需要传入一个 ChannelInitializer 通道初始化类的实例作为参数。每当父通道成功接收一个连接，并创建成功一个子通道后，就会初始化子通道，此时这里配置的 ChannelInitializer 实例就会被调用。
在 ChannelInitializer 通道初始化类的实例中，有一个 initChannel 初始化方法，在子通道创建后会被执行到，向子通道流水线增加业务处理器。
```java
// 装配子通道流水线  
bootstrap.childHandler(new ChannelInitializer<SocketChannel>() {  
    // 有连接到达的时候会创建一个通道  
    @Override  
    protected void initChannel(SocketChannel ch) throws Exception {  
        // 流水线：负责管理通道中的 Handler 处理器  
        // 向子通道流水线添加一个 Handler 处理器  
        ch.pipeline().addLast(new NettyDiscardHandler());  
    }  
});
```
>[!question] 为什么仅装配子通道的流水线，而不需要装配父通道的流水线呢？
原因是：父通道也就是 NioServerSocketChannel 的内部业务处理是固定的：接受新连接后，创建子通道，然后初始化子通道，所以不需要特别的配置，由 Netty 自行进行装配。当然，如果需要完成特殊的父通道业务处理，可以类似的使用ServerBootstrap 的 handler(ChannelHandler handler) 方法，为父通道设置 ChannelInitializer 初始化器。

在装配流水线时需要注意的是，ChannelInitializer 处理器有一个泛型参数 SocketChannel，它代表 **需要初始化的通道类型**，这个类型需要和前面的引导类中设置的传输通道类型保持对应。

### 绑定服务器的监听端口
```java
// step6：开始绑定端口，通过调用 sync 同步方法阻塞直到绑定成功
ChannelFuture channelFuture = b.bind().sync();
Logger.info(" 服务器启动成功，监听端口: " + channelFuture.channel().localAddress());
```
b.bind() 方法的功能：返回一个端口绑定 Netty 的异步任务 channelFuture。在这里，并没有给 channelFuture 异步任务增加回调监听器，而是阻塞 channelFuture 异步任务，直到端口绑定任务执行完成。
在 Netty 中，所有的 IO 操作都是异步执行的，这就意味着任何一个IO操作会立刻返回，在返回的时候，异步任务还没有真正执行。什么时候执行完成呢？Netty 中的 IO 操作，都会返回异步任务实例（如 ChannelFuture 实例）。通过该异步任务实例，既可以实现同步阻塞一直到 ChannelFuture 异步任务执行完成，也可以为其增加事件监听器的方式注册异步回调逻辑，以获得 Netty 中的 IO 操作的真正结果。而上面所使用的，是同步阻塞一直到 ChannelFuture 异步任务执行完成的处理方式。

### 自我阻塞直到监听通道关闭
```java
// step7：自我阻塞，直到通道关闭的异步任务结束
ChannelFuture closeFuture = channelFuture.channel().closeFuture();
closeFuture.sync();
```
如果要阻塞当前线程直到通道关闭，可以使用通道的closeFuture()方法，以获取通道关闭的异步任务。当通道被关闭时，closeFuture实例的sync()方法会返回。

### 关闭 EventLoopGroup
```java
// step8：释放掉所有资源，包括创建的反应器线程
workerLoopGroup.shutdownGracefully();
bossLoopGroup.shutdownGracefully();
```
关闭 Reactor 反应器轮询组，同时会关闭内部的 SubReactor 子反应器线程，也会关闭内部的 Selector 选择器、内部的轮询线程以及负责查询的所有的子通道。在子通道关闭后，会释放掉底层的资源，如 Socket 文件描述符等。
# Channel Option 通道选项
无论是对于 NioServerSocketChannel 父通道类型，还是对于 NioSocketChannel 子通道类型，都可以设置一系列的 ChannelOption 通道选项。ChannelOption 类中定义了一系列的选项，下面介绍一些常见的选项：

### SO_RCVBUF 和 SO_SNDBUF：接受与发送缓冲区
此为 TCP 传输选项，每个 TCP socket（套接字）在内核中都有一个发送缓冲区和一个接收缓冲区，这两个选项就是用来设置TCP连接的这两个缓冲区大小的。
TCP 的全双工工作模式以及 TCP 的滑动窗口对两个独立的缓冲区都有依赖。

### TCP_NODELAY：延时发送
此为TCP传输选项，如果设置为 true 表示立即发送数据。TCP_NODELAY 就是用于启用或关闭 Nagle 算法。
	如果要求高实时性，有数据发送时就马上发送，就将该选项设置为true（关闭Nagle算法）；
	如果要减少发送次数减少网络交互，就设置为 false（启用Nagle算法），等累积一定大小的数据后再发送。
TCP_NODELAY 的值 Netty 默认为 true，而操作系统默认为 False。Nagle 算法将小的碎片数据连接成更大的报文（或数据包）来最小化所发送报文的数量，如果需要发送一些较小的报文，则需要禁用该算法。
Netty 默认禁用 Nagle 算法，报文会立即发送出去，从而最小化报文传输的延时。

### SO_KEEPALIVE：心跳机制
此为 TCP 传输选项，表示是否开启 TCP 协议的心跳机制。true 为连接保持心跳，默认值为 false。启用该功能时，TCP 会主动探测空闲连接的有效性。
可以将此功能视为 TCP 的心跳机制，需要注意的是：默认的心跳间隔是 7200 秒即 2 小时。Netty 默认关闭该功能。

### SO_REUSEADDR：地址复用
此为 TCP 传输选项，如果为 true 时表示地址复用，默认值为 false。有四种情况需要用到这个参数设置：
	当有一个地址和端口相同的连接 socket1 处于 TIME_WAIT 状态时，而又希望启动一个新的连接 socket2 要占用该地址和端口；
	有多块网卡或用 IP Alias 技术的机器在同一端口启动多个进程，但每个进程绑定的本地IP地址不能相同；
	同一进程绑定相同的端口到多个 socket（套接字）上，但每个socket绑定的IP地址不同；
	完全相同的地址和端口的重复绑定，但这只用于 UDP 的多播，不用于 TCP。

### SO_LINGER：close 行为控制
此为TCP传输选项，选项可以用来控制 socket.close() 方法被调用后的行为，包括延迟关闭时间。
	如果此选项设置为 -1，表示 socket.close() 方法在调用后立即返回，但操作系统底层会将发送缓冲区的数据全部发送到对端；
	如果此选项设置为0，表示 socket.close() 方法在调用后会立即返回，但是操作系统会放弃发送缓冲区数据，而是直接向对端发送RST包，对端将收到复位错误；
	如果此选项设置为非0整数值，表示调用 socket.close() 方法的线程被阻塞，直到延迟时间到来，发送缓冲区中的数据发送完毕，若超时，则对端会收到复位错误。
SO_LINGER的默认值为-1，表示禁用该功能。

### SO_BACKLOG：TCP 队列大小设置
此为 TCP 传输选项，表示服务器端接收连接的队列长度，如果队列已满，客户端连接将被拒绝。
服务端在处理客户端新连接请求时（三次握手），是顺序处理的，所以同一时间只能处理一个客户端连接，多个客户端来的时候，服务端将不能处理的客户端连接请求放在队列中等待处理，队列的大小通过SO_BACKLOG 指定。具体来说，服务端对完成第二次握手的连接放在一个队列（暂时称A队列），如果进一步完成第三次握手，再把的连接从 A 队列移动到新队列（暂时称 B 队列），接下来应用程序会通过 accept 方法取出握手成功的连接，而系统则会将该连接从 B 队列移除。 
A 队列和 B 队列的长度之和是 SO_BACKLOG 指定的值，当 A 和 B 队列的长度之和大于 SO_BACKLOG 值时，新连接将会被 TCP 内核拒绝，所以，如果 SO_BACKLOG 过小，可能会出现 accept 速度跟不上，A 和 B 两队列满了，导致新客户端无法连接。

### SO_BROADCAST：设置为广播模式
此为TCP传输选项，表示设置为广播模式。