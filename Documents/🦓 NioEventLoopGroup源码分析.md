# 模板代码

>[!info] NioEventLoopSource
```java
public class NioEventLoopSource {  
    private static final int PORT = 8080;  
    public static void main(String[] args) {  
        EventLoopGroup parentGroup = new NioEventLoopGroup();  
        EventLoopGroup childGroup = new NioEventLoopGroup();  
        try {  
            ServerBootstrap b = new ServerBootstrap();  
            b.group(parentGroup, childGroup)  
                    .channel(NioServerSocketChannel.class)  
                    .option(ChannelOption.SO_BACKLOG, 128)  
                    .childHandler(new MyChannelInitializer());  
            ChannelFuture f = b.bind(PORT).sync();  
            f.channel().closeFuture().sync();  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        } finally {  
            childGroup.shutdownGracefully();  
            parentGroup.shutdownGracefully();  
        }  
    }  
}
```

>[!info] MyChannelInitializer
```java
public class MyChannelInitializer extends ChannelInitializer<NioSocketChannel> {  
    @Override  
    protected void initChannel(NioSocketChannel nioSocketChannel) throws Exception {  
        ChannelPipeline pipeline = nioSocketChannel.pipeline();  
        pipeline.addLast(new ChannelInboundHandlerAdapter() {  
            @Override  
            public void channelActive(ChannelHandlerContext ctx) throws Exception {  
                System.out.println("Handler1: channelActive");  
            }  
        });  
        pipeline.addLast(new ChannelInboundHandlerAdapter() {  
            @Override  
            public void channelActive(ChannelHandlerContext ctx) throws Exception {  
                System.out.println("Handler2: channelActive");  
            }  
        });  
    }  
}
```

# 类结构树
![[EventLoopGroup 继承关系.png]]
`NioEventLoopGroup` 通过实现 JUC 的方法，来实现自己的相关功能。

# EventExecutorGroup
`EventExecutorGroup` 使用 `next()` 方法负责提供 `EventExecutor`。除此之外，它还负责处理生命周期，并且可以以一种全局的方式进行关闭。
```java
/**
 * The {@link EventExecutorGroup} is responsible for providing the {@link EventExecutor}'s to use
 * via its {@link #next()} method. Besides this, it is also responsible for handling their
 * life-cycle and allows shutting them down in a global fashion.
 */
public interface EventExecutorGroup extends ScheduledExecutorService, Iterable<EventExecutor> {

	...

    /**
     * Returns one of the {@link EventExecutor}s managed by this {@link EventExecutorGroup}.
     */
    EventExecutor next();

	...
}

```
`EventExecutorGroup.next()` 返回一个由 `EventExecutorGroup` 管理的事件执行器。组里包含了若干个 `EventExecutor`。
# EventLoopGroup
EventLoopGroup 继承 EventExecutorGroup的接口。
EventLoopGroup 本身是特殊的 EventExecutorGroup，它的作用是会在事件循环（处理链接、输入输出消息等）的过程当中，进行 selection 操作当中允许注册一个一个的 channel 链接。
>[!info] EventLoopGroup.java
```java
/**
 * Special {@link EventExecutorGroup} which allows registering {@link Channel}s that get
 * processed for later selection during the event loop.
 *
 */
public interface EventLoopGroup extends EventExecutorGroup {
    /**
     * Return the next {@link EventLoop} to use
     */
    @Override
    EventLoop next();

    /**
     * Register a {@link Channel} with this {@link EventLoop}. The returned {@link ChannelFuture}
     * will get notified once the registration was complete.
     */
    ChannelFuture register(Channel channel);

    /**
     * Register a {@link Channel} with this {@link EventLoop} using a {@link ChannelFuture}. The passed
     * {@link ChannelFuture} will get notified once the registration was complete and also will get returned.
     */
    ChannelFuture register(ChannelPromise promise);

    /**
     * Register a {@link Channel} with this {@link EventLoop}. The passed {@link ChannelFuture}
     * will get notified once the registration was complete and also will get returned.
     *
     * @deprecated Use {@link #register(ChannelPromise)} instead.
     */
    @Deprecated
    ChannelFuture register(Channel channel, ChannelPromise promise);
}
```

| **方法名称**                                                           | **方法解释**                                                                                                   |
| ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| `EventLoopGroup.next()`                                            | 返回下一个事件循环                                                                                                  |
| `EventLoopGroup.register(Channel channel)`                         | 将一个 `Channel` 注册到事件循环当中，所返回的 `ChannelFuture` 在注册完成之后就会收到一个通知。                                              |
| `EventLoopGroup.register(ChannelPromise promise)`                  | 与上面的方法构成一个重载，`ChannelPromise` 里面继承了 `ChannelFuture`，里面包含了 `channel`。在注册完成之后 `ChannelFuture` 会收到一个通知并且也会返回。 |
| `EventLoopGroup.register(Channel channel, ChannelPromise promise)` | 因为ChannelPromise已经包含了Channel，方法重复了所以不被推荐使用了。                                                               |
上面提到的 `ChannelFuture` 是一个异步方法，`ChannelFuture` 是继承自 jdk 1.5 里面的 `Future` 方法。
 









