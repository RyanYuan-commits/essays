## 1 基本介绍

在 `Reactor` 反应器经典模型中, 反应器查询到 IO 事件后, 分发到 `Handler` 业务处理器, 由 `Handler` 完成IO操作和业务处理. 

整个的 IO 处理操作环节大致包括: 从通道读数据包、数据包解码、业务处理、目标数据编码、把数据包写到通道, 然后由通道发送到对端. 从通道读数据包和由通道发送到对端由 Netty 的底层负责, 不需要用户处理.

前面已经介绍过, 从应用程序开发人员的角度来看, 有入站和出站两种类型操作. 
	
- **入站处理触发的方向**: 自底向上, Netty 通道到 `ChannelInboundHandler` 入站处理器. 
- **出站处理触发的方向**: 自顶向下, 从 `ChannelOutboundHandler` 出站处理器到 Netty 的通道. 

按照这种触发方向来区分, IO 处理操作环节前面的数据包解码、业务处理两个环节——属于入站处理器的工作；后面目标数据编码、把数据包写到通道中两个环节——属于出站处理器的工作. 

## 2 ChannelInboundHandler 入站处理器

当对端数据入站到 Netty 通道时, Netty 将触发入站处理器 ChannelInboundHandler 所对应的入站API, 进行入站操作处理. 

对于 `ChannelInboundHandler` 的核心方法, 大致的介绍如下: 

- `channelRegistered`:当通道注册完成后, Netty会调用 fireChannelRegistered 方法, 触发通道注册事件. 而在通道流水线注册过的入站处理器 Handler 的 channelRegistered 回调方法, 将会被调用到. 
- `channelActive`: 当通道激活完成后, Netty会调用fireChannelActive方法, 触发通道激活事件. 而在通道流水线注册过的入站处理器的 channelActive 回调方法, 会被调用到. 
- `channelRead`: 当通道缓冲区可读, Netty的反应器完成数据读取后, 会调用 fireChannelRead 发射读取到的二进制数据. 而在通道流水线注册过的入站处理器的channelRead回调方法, 会被调用到, 以便完成入站数据的读取和处理. 
- `channelReadComplete`: 当通道缓冲区读完, Netty 会调用 fireChannelReadComplete, 触发通道缓冲区读完事件. 而在通道流水线注册过的入站处理器的 channelReadComplete 回调方法, 会被调用到. 
- `channelInactive`: 当连接被断开或者不可用时, Netty 会调用 fireChannelInactive, 触发连接不可用事件. 而在通道流水线注册过的入站处理器的 channelInactive 回调方法, 会被调用到. 
- `exceptionCaught`: 当通道处理过程发生异常时, Netty 会调用 fireExceptionCaught, 触发异常捕获事件. 而通道流水线注册过的入站处理器的 exceptionCaught 方法, 会被调用到. 注意, 这个方法是在通道处理器中 ChannelHandler 定义的方法, 入站处理器、出站处理器接口都继承到了该方法. 

上面介绍的并不是 `ChanneInboundHandler` 的全部方法, 仅仅介绍了其中几种比较重要的方法. 在 Netty 中, 入站处理器的默认实现为 `ChannelInboundHandlerAdapter`, 在实际开发中, 只需要继承这个 `ChannelInboundHandlerAdapter` 默认实现, 重写自己需要的回调方法即可. 

## 3 ChannelOutboundHandler 出站处理器

当业务处理完成后, 需要操作 Java NIO 底层通道时, 通过一系列的 `ChannelOutboundHandler` 出站处理器, 完成 Netty 通道到底层通道的操作. 比方说建立底层连接、断开底层连接、写入底层 Java NIO 通道等. `ChannelOutboundHandler` 接口定义了大部分的出站操作. 

- `bind`: 监听地址(IP+端口)绑定: 完成底层 Java IO 通道的IP地址绑定. 如果使用 TCP 传输协议, 这个方法用于服务器端. 
- `connect`: 连接服务端: 完成底层 Java IO 通道的服务器端的连接操作. 如果使用 TCP 传输协议, 这个方法用于客户端. 
- `write`: 写数据到底层: 完成 Netty 通道向底层 Java IO 通道的数据写入操作. 此方法仅仅是触发一下操作而已, 并不是完成实际的数据写入操作. 
- `flush`: 将底层缓存区的数据腾空, 立即写出到对端. 
- `read`: 出站处理的 read 操作, 是启动数据读取, 或者说开始数据的读取操作, 不是实际的数据读取. 只有入站处理的 read 操作, 才真正执行底层读数据. 入站 read 处理在完成 Netty 通道从 Java IO 通道的数据读取后, 再把数据发射到通道的 pipeline, 最后数据会依次进入 pipeline 的各个入站处理器, 最终被入站处理器的 channelRead 方法处理. 
- `disConnect`: 断开服务器连接: 断开底层 Java IO 通道的 socket 连接. 如果使用 TCP 传输协议, 此方法主要用于客户端. 
- `close`: 主动关闭通道: 关闭底层的通道, 例如服务器端的新连接监听通道. 
## 4 ChannelInitializer 通道初始化处理器

在前面已经讲到, Channel 通道和 Handler 业务处理器的关系是: 一条 Netty 的通道拥有一条 Handler 业务处理器流水线, 负责装配自己的 Handler 业务处理器. 

装配 Handler 的工作, 发生在通道开始工作之前. 现在的问题是: 如果向流水线中装配业务处理器呢？这就得借助通道的初始化处理器——ChannelInitializer. 

```java
ChannelInitializer<EmbeddedChannel> channelInitializer = new ChannelInitializer<EmbeddedChannel>() {  
    @Override  
    protected void initChannel(EmbeddedChannel ch) {  
        ch.pipeline().addLast(new Handlers.Handler1());  
        ch.pipeline().addLast(new Handlers.Handler2());  
        ch.pipeline().addLast(new Handlers.Handler3());  
    }  
};
```

上面的 ChannelInitializer 也是通道初始化器, 属于入站处理器的类型. initChannel() 方法是 ChannelInitializer 定义的一个抽象方法, 这个抽象方法需要开发人员自己实现. 

在通道初始化时, 会调用提前注册的初始化处理器的 initChannel(…) 方法. 比如, **在父通道接收到新连接并且要初始化其子通道时**, 会调用初始化器的 initChannel(…) 方法, 并且会将新接收的通道作为参数, 传递给此方法. 

一般来说, initChannel() 方法的大致业务代码是: 拿到新连接 pipeline 作为实际参数, 往它的流水线中装配Handler业务处理器. 

## 5 ChannelInboundHandler 生命周期

>[!info] InHandlerDemo: 实现一个简单的 InHandlerDemo, 让他继承并重写 ChannelInboundHandlerAdapter 的所有方法, 在重写的方法中只是简单的调用其父类的方法, 并做简单的输出操作. 
```java
public class InHandlerDemo extends ChannelInboundHandlerAdapter {  
  
    @Override  
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {  
        System.out.println("exceptionCaught() 被调用");  
        super.exceptionCaught(ctx, cause);  
    }  
  
    @Override  
    public void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception {  
        System.out.println("channelWritabilityChanged() 被调用");  
        super.channelWritabilityChanged(ctx);  
    }  
  
    @Override  
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {  
        System.out.println("userEventTriggered() 被调用");  
        super.userEventTriggered(ctx, evt);  
    }  
  
    @Override  
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {  
        System.out.println("channelReadComplete() 被调用");  
        super.channelReadComplete(ctx);  
    }  
  
    @Override  
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {  
        System.out.println("channelRead() 被调用");  
        super.channelRead(ctx, msg);  
    }  
  
    @Override  
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {  
        System.out.println("channelInactive() 被调用");  
        super.channelInactive(ctx);  
    }  
  
    @Override  
    public void channelActive(ChannelHandlerContext ctx) throws Exception {  
        System.out.println("channelActive() 被调用");  
        super.channelActive(ctx);  
    }  
  
    @Override  
    public void channelUnregistered(ChannelHandlerContext ctx) throws Exception {  
        System.out.println("channelUnregistered() 被调用");  
        super.channelUnregistered(ctx);  
    }  
  
    @Override  
    public void channelRegistered(ChannelHandlerContext ctx) throws Exception {  
        System.out.println("channelRegistered() 被调用");  
        super.channelRegistered(ctx);  
    }  
  
}
```

>[!info] 使用嵌入式通道来测试 Handler
```java
public class InHandlerDemoTester {  
  
    public static void main(String[] args) {  
        InHandlerDemo inHandlerDemo = new InHandlerDemo();  
        ChannelInitializer<EmbeddedChannel> channelInitializer = new ChannelInitializer<EmbeddedChannel>() {  
            @Override  
            protected void initChannel(EmbeddedChannel ch) throws Exception {  
                ch.pipeline().addLast(inHandlerDemo);  
            }  
        };  
        EmbeddedChannel embeddedChannel = new EmbeddedChannel(channelInitializer);  
        ByteBuf buffer = ByteBufAllocator.DEFAULT.buffer();  
        // 调用写入方法
        embeddedChannel.writeInbound(buffer);  
        embeddedChannel.flush();  
        embeddedChannel.close();  
    }  
  
}
```

最终的输出结果: 

```
channelRegistered() 被调用
channelActive() 被调用
channelRead() 被调用
channelReadComplete() 被调用
channelInactive() 被调用
channelUnregistered() 被调用
```

Channel 中回调方法的执行顺序是这样的: channelRegistered() => channelActive() => 数据的入站回调方法 => channelInactive() => channelUnregistered()
其中, 数据的入站回调方法为: channelRead() => channelReadComplete()

除了两个入站回调方法之外, 其余的几个方法都是和 ChannelHandler 的生命周期相关的, 具体来说: 
- `channelRegistered`: 当通道成功绑定一个 `NioEventLoop Reactor` 后, 此方法将被回调. 
- `channelActive`: 当通道激活成功后, 此方法将被回调. 通道激活成功指的是, 所有的业务处理器添加、注册的异步任务完成, 并且与 `NioEventLoop` 反应器绑定的异步任务完成. 
- `channelInactive`: 当通道的底层连接已经不是 ESTABLISH 状态, 或者底层连接已经关闭时, 会首先回调所有业务处理器的 `channelInactive` 方法. 
- `channelUnregistered`: 通道和 NioEventLoop 反应器解除绑定, 移除掉对这条通道的事件处理之后, 回调所有业务处理器的 `channelUnregistered` 方法. 
- `handlerRemoved`: 最后, Netty 会移除掉通道上所有的业务处理器, 并且回调所有的业务处理器的 `handlerRemoved` 方法. 

数据传输的入站回调方法, 对于入站处理器, 有两个很重要的回调方法: 

- `channelRead`: 有数据包入站, 通道可读. 流水线会启动入站处理流程, 从前向后, 入站处理器的 `channelRead` 方法会被依次回调到. 
- `channelReadComplete`: 流水线完成入站处理后, 会从前向后, 依次回调每个入站处理器的 `channelReadComplete` 方法, 表示数据读取完毕. 