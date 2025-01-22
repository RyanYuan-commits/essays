# 第一个 Netty 服务端程序
```java
public class NettyDiscardServer {  
    public static void main(String[] args) {  
        ServerBootstrap bootstrap = new ServerBootstrap();  
        NioEventLoopGroup bossLoopGroup = new NioEventLoopGroup();  
        NioEventLoopGroup workerLoopGroup = new NioEventLoopGroup();  
  
        try {  
            // 设置反应器轮训组  
            bootstrap.group(bossLoopGroup, workerLoopGroup);  
            // 设置 NIO 类型的通道  
            bootstrap.channel(NioServerSocketChannel.class);  
            // 设置监听端口  
            bootstrap.localAddress(8080);  
            // 设置通道的参数  
            bootstrap.option(ChannelOption.SO_KEEPALIVE, true);  
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
            // 开始绑定服务器  
            // 通过 sync 同步阻塞知道绑定成功  
            ChannelFuture channelFuture = bootstrap.bind().sync();  
            // 等待通道关闭的异步任务结束  
            // 服务监听通道会一直等待通道关闭的异步任务结束  
            ChannelFuture closeFuture = channelFuture.channel().closeFuture();  
            closeFuture.sync();  
        } catch (InterruptedException e) {  
            throw new RuntimeException(e);  
        } finally {  
            bossLoopGroup.shutdownGracefully();  
            workerLoopGroup.shutdownGracefully();  
        }  
    }  
}
```
首先要说的是反应器模式中的 Reactor 反应器组件。[[🌲【并发】Reactor 反应器模式]]，**反应器组件的作用是进行 IO 事件轮训，以及将事件分配到合适的 Handler**。
在上面的例子中，使用了两个NioEventLoopGroup反应器组件实例。
	第一个负责服务器通道新连接的IO事件的监听，可以形象的理解为“包工头”角色。
	第二个主要负责传输通道的IO事件的处理和数据传输，可以形象的理解为“工人”角色。

其次要说的是反应器模式中的 Handler（处理器）角色组件。**Handler 处理器的作用是对应到 IO 事件，完成 IO 事件的业务处理**。

再次，在上面的例子中还用到了 Netty 的服务引导类 ServerBootstrap。服务引导类是一个组装和集成器，它的职责它的职责将不同的 Netty 组件组装在一起。
此外 ServerBootstrap 能够按照应用场景的需要，为组件设置好基础性的参数，最后帮助快速实现 Netty 服务器的监听和启动。
# 业务处理器 NettyDiscardHandler
```java
public class NettyDiscardHandler extends ChannelInboundHandlerAdapter {  
  
    @Override  
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {  
        ByteBuf in = (ByteBuf) msg;  
        try {  
            while (in.isReadable()) {  
                System.out.println((char) in.readByte());  
            }  
            System.out.println();  
        } finally {  
            // 释放资源  
            ReferenceCountUtil.release(msg);  
        }  
    }  
      
}
```
Netty 的 Handler 处理器需要处理多种 IO 事件（如读就绪、写就绪），对应于不同的IO事件，Netty提供了一些基础的方法。这些方法都已经提前封装好，应用程序直接继承或者实现即可。比如说，对于处理入站的IO事件，其对应的接口为 ChannelInboundHandler 入站处理接口，并且 Netty 提供了 ChannelInboundHandlerAdapter 适配器作为入站处理器的默认实现。
