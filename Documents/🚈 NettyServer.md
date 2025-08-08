# Hello Netty Server
>[!info] NettyServer.java
```java
public class NettyServer {  
    public static void main(String[] args) {  
        EventLoopGroup bossGroup = new NioEventLoopGroup();  
        EventLoopGroup workerGroup = new NioEventLoopGroup();  
        try {  
            ChannelFuture bind = new ServerBootstrap()  
                    .group(bossGroup, workerGroup)  
                    .channel(NioServerSocketChannel.class)  
                    .option(ChannelOption.SO_BACKLOG, 128)  
                    .childHandler(new ChannelInitializer<NioSocketChannel>() {  
                        @Override  
                        protected void initChannel(NioSocketChannel ch) throws Exception {  
                            ch.pipeline().addLast(new StringDecoder());  
                            ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {  
                                @Override  
                                public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {  
                                    System.out.println(msg);  
                                }  
                            });  
                            System.out.println("initChannel");  
                        }  
                    }).bind(8080);  
            ChannelFuture channelFuture = bind.sync();  
            channelFuture.channel().closeFuture().sync();  
        } catch (Exception e) {  
            System.out.println(e.getMessage());  
        } finally {  
            bossGroup.shutdownGracefully();  
            workerGroup.shutdownGracefully();  
        }  
    }  
}
```

# Netty Server 接收数据
>[!info] NettyServer.java
```java
public class NettyServer {  
    public static void main(String[] args) {  
        EventLoopGroup bossGroup = new NioEventLoopGroup();  
        EventLoopGroup workerGroup = new NioEventLoopGroup();  
        try {  
            ChannelFuture bind = new ServerBootstrap()  
                    .group(bossGroup, workerGroup)  
                    .channel(NioServerSocketChannel.class)  
                    .option(ChannelOption.SO_BACKLOG, 128)  
                    .childHandler(new MyChannelInitializer())  
                    .bind(8080);  
            ChannelFuture channelFuture = bind.sync();  
            channelFuture.channel().closeFuture().sync();  
        } catch (Exception e) {  
            System.out.println(e.getMessage());  
        } finally {  
            bossGroup.shutdownGracefully();  
            workerGroup.shutdownGracefully();  
        }  
    }  
}
```

>[!info] MyChannelInitializer.java
```java
public class MyChannelInitializer extends ChannelInitializer<NioSocketChannel> {  
    @Override  
    protected void initChannel(NioSocketChannel ch) throws Exception {  
        // log something  
        System.out.println("[Log]: a client connected successfully");  
        System.out.println("[Log]:Client ip = " + ch.localAddress().getHostString());  
        System.out.println("[Log]:Client port = " + ch.localAddress().getPort());  
        System.out.println("====================================");  
  
        ch.pipeline().addLast(new MyServerHandler());  
    }  
}
```

>[!info] MyServerHandler.java
```java
public class MyServerHandler extends ChannelInboundHandlerAdapter {  
    @Override  
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {  
        ByteBuf byteBuf = (ByteBuf)msg;  
        byte[] msgByte = new byte[byteBuf.readableBytes()];  
        byteBuf.readBytes(msgByte);  
        System.out.print(new Date() + "接收到消息：");  
        System.out.println(new String(msgByte, Charset.forName("GBK")));  
        super.channelRead(ctx, msg);  
    }  
}
```

# Netty Server 字符串解码器
在实际开发中，server 端接收数据后我们希望他是一个字符串或者是一个对象类型，而不是字节码，那么；
1. 在netty中是否可以自动的把接收的 Bytebuf 数据转String，不需要我手动处理？ 答；有，可以在管道中添加一个StringDecoder。
2. 在网络传输过程中有半包粘包的问题，netty能解决吗？ 答：能，**netty提供了很丰富的解码器，在正确合理的使用下就能解决半包粘包问题**。
3. 常用的 String 字符串下有什么样的解码器呢？ 答：不仅在 String下有处理半包粘包的解码器在处理其他的数据格式也有，其中谷歌的 protobuf 数据格式就是其中一个。对于String 的有以下常用的三种：
    1. LineBasedFrameDecoder 基于换行
    2. DelimiterBasedFrameDecoder 基于指定字符串
    3. FixedLengthFrameDecoder 基于字符串长度

>[!info] MyChannelInitializer.java
```java
public class MyChannelInitializer extends ChannelInitializer<NioSocketChannel> {  
    @Override  
    protected void initChannel(NioSocketChannel ch) {  
        System.out.println("====================================");  
        /* 解码器 */        // 基于换行符号  
        ch.pipeline().addLast(new LineBasedFrameDecoder(1024));  
  
        // 基于指定字符串【换行符，这样功能等同于LineBasedFrameDecoder】  
        // e.pipeline().addLast(new DelimiterBasedFrameDecoder(1024, false, Delimiters.lineDelimiter()));  
  
        // 基于最大长度  
        // e.pipeline().addLast(new FixedLengthFrameDecoder(4));  
  
        // 解码转String，注意调整自己的编码格式GBK、UTF-8  
        ch.pipeline().addLast(new StringDecoder(Charset.forName("GBK")));  
        ch.pipeline().addLast(new MyServerHandler());  
    }  
}
```

>[!info] MyChannelInitializer.java
```java
public class MyServerHandler extends ChannelInboundHandlerAdapter {  
    @Override  
    public void channelActive(ChannelHandlerContext ctx) throws Exception {  
        SocketChannel channel = (SocketChannel) ctx.channel();  
        System.out.println("[Log]: a client connect successfully");  
        System.out.println("[Log]:Client ip = " + channel.localAddress().getHostString());  
        System.out.println("[Log]:Client port = " + channel.localAddress().getPort());  
        System.out.println("====================================");  
        super.channelActive(ctx);  
    }  
  
    @Override  
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {  
        System.out.println("[Log]: Message = " + msg);  
        super.channelRead(ctx, msg);  
    }  
}
```
此时的 Handler 就不需要再进行解码了，因为经过上面的解码器，传到 Handler 是一个字符串。
我们使用客户端发送这样的数据
```java
"123456\n123"
```
经过 `LineBasedFrameDecoder` 的解码，会分为两个 `123456` 和 `123` 字符，分别进行 GBK 解码，然后到达 Handler 输出，所以，我们最终会得到两个结果：
```java
[Log]: Message = 123456
[Log]: Message = 123
```
# Netty Server 编解码器与数据收发
本章节主要介绍服务端在收到数据后，通过writeAndFlush发送ByteBuf字节码向客户端传输信息。
因为我们使用客户端模拟器的编码是GBK格式，所以代码中也需要将字节码转换为GBK，否则会乱码。
netty通信就向一个流水channel管道，我们可以在管道的中间插入一些‘挡板’为我们服务。比如字符串的编码解码，在前面我们使用new StringDecoder(Charset.forName("GBK"))进行字符串解码，这样我们在收取数据就不需要手动处理字节码。
那么本章节我们使用与之对应的new StringEncoder(Charset.forName("GBK"))进行进行字符串编码，用以实现服务端在发送数据的时候只需要传输字符串内容即可。
>[!info] NettyServer
```java
public class NettyServer {  
    public static void main(String[] args) {  
        EventLoopGroup bossGroup = new NioEventLoopGroup();  
        EventLoopGroup workerGroup = new NioEventLoopGroup();  
        try {  
            ChannelFuture bind = new ServerBootstrap()  
                    .group(bossGroup, workerGroup)  
                    .channel(NioServerSocketChannel.class)  
                    .option(ChannelOption.SO_BACKLOG, 128)  
                    .childHandler(new MyChannelInitializer())  
                    .bind(8080);  
            ChannelFuture channelFuture = bind.sync();  
            channelFuture.channel().closeFuture().sync();  
        } catch (Exception e) {  
            System.out.println(e.getMessage());  
        } finally {  
            bossGroup.shutdownGracefully();  
            workerGroup.shutdownGracefully();  
        }  
    }  
}
```

>[!info] MyChannelInitializer
```java
public class MyChannelInitializer extends ChannelInitializer<NioSocketChannel> {  
    @Override  
    protected void initChannel(NioSocketChannel ch) {  
        System.out.println("====================================");  
        // Decoders  
        // Decoder that splits the received ByteBuf        ch.pipeline().addLast(new LineBasedFrameDecoder(1024));  
        // Decoder that decodes the received ByteBuf into a String  
        ch.pipeline().addLast(new StringDecoder(Charset.forName("GBK")));  
        ch.pipeline().addLast(new StringEncoder(Charset.forName("GBK")));  
        ch.pipeline().addLast(new MyServerHandler());  
    }  
}
```

>[!info] MyServerHandler
```java
public class MyServerHandler extends ChannelInboundHandlerAdapter {  
    @Override  
    public void channelActive(ChannelHandlerContext ctx) throws Exception {  
        SocketChannel channel = (SocketChannel) ctx.channel();  
        System.out.println("[Log]: a client connect successfully");  
        System.out.println("[Log]:Client ip = " + channel.localAddress().getHostString());  
        System.out.println("[Log]:Client port = " + channel.localAddress().getPort());  
        System.out.println("====================================");  
        // Notice client connected  
        String msg = "Welcome to Netty Server";  
        ctx.writeAndFlush(msg);  
        super.channelActive(ctx);  
    }  
  
    @Override  
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {  
        System.out.println("[Log]: Message = " + msg);  
        ctx.writeAndFlush("Message received successfully");  
        super.channelRead(ctx, msg);  
    }  
  
    @Override  
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {  
        System.out.println("[Log]: Client disconnected");  
        super.channelInactive(ctx);  
    }  
  
    @Override  
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {  
        ctx.close();  
        System.out.println("异常信息：\r\n" + cause.getMessage());  
    }  
}
```

>[!info] NettyClient
```java
public class NettyClient {  
    public static void main(String[] args) throws InterruptedException {  
        Channel channel = new Bootstrap()  
                .group(new NioEventLoopGroup())  
                .channel(NioSocketChannel.class)  
                .handler(new ChannelInitializer<NioSocketChannel>() {  
                    @Override  
                    protected void initChannel(NioSocketChannel ch) {  
                        ch.pipeline().addLast(new StringDecoder(Charset.forName("GBK")));  
                        ch.pipeline().addLast(new StringEncoder(Charset.forName("GBK")));  
                        ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {  
                            @Override  
                            public void channelRead(ChannelHandlerContext ctx, Object msg) {  
                                System.out.println(msg);  
                            }  
                        });  
                    }  
                })  
                .connect("localhost", 8080)  
                .sync()  
                .channel();  
  
        // Get user input  
        new Thread(() -> {  
            Scanner scanner = new Scanner(System.in);  
            String message;  
            while (!"quit".equals(message = scanner.nextLine())) {  
                channel.writeAndFlush(message + "\n");  
            }  
        }).start();  
    }    
}
```
# Netty Server 群发消息
>[!info] ChannelHandler 用于存放用户的 Channel 信息。
```java
public class ChannelHandler {  
    // storage all user channel  
    public static ChannelGroup channelGroup = new DefaultChannelGroup(GlobalEventExecutor.INSTANCE);  
}
```

>[!info]
```java
public class MyChannelInitializer extends ChannelInitializer<NioSocketChannel> {  
    @Override  
    protected void initChannel(NioSocketChannel ch) {  
        System.out.println("====================================");  
        // Decoders  
        // Decoder that splits the received ByteBuf        ch.pipeline().addLast(new LineBasedFrameDecoder(1024));  
        // Decoder that decodes the received ByteBuf into a String  
        ch.pipeline().addLast(new StringDecoder(Charset.forName("GBK")));  
        ch.pipeline().addLast(new StringEncoder(Charset.forName("GBK")));  
        ch.pipeline().addLast(new MyServerHandler());  
    }  
}
```

>[!info] MyServerHandler
```java
public class MyServerHandler extends ChannelInboundHandlerAdapter {  
    @Override  
    public void channelActive(ChannelHandlerContext ctx) throws Exception {  
        // Add to channelGroup  
        ChannelHandler.channelGroup.add(ctx.channel());  
        SocketChannel channel = (SocketChannel) ctx.channel();  
        System.out.println("[Log]: a client connect successfully");  
        System.out.println("[Log]:Client ip = " + channel.localAddress().getHostString());  
        System.out.println("[Log]:Client port = " + channel.localAddress().getPort());  
        System.out.println("====================================");  
        // Notice client connected  
        String msg = "Welcome to Netty Server";  
        ctx.writeAndFlush(msg);  
        super.channelActive(ctx);  
    }  
  
    @Override  
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {  
        System.out.println("[Log]: Message = " + msg);  
//        ctx.writeAndFlush("Message received successfully");  
        // Broadcast to all clients        
        ChannelHandler.channelGroup.writeAndFlush(msg);  
        super.channelRead(ctx, msg);  
    }  
  
    @Override  
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {  
        System.out.println("[Log]: Client disconnected");  
        ChannelHandler.channelGroup.remove(ctx.channel());  
        super.channelInactive(ctx);  
    }  
  
    @Override  
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {  
        ctx.close();  
        System.out.println("异常信息：\r\n" + cause.getMessage());  
    }  
}
```

>[!info] NettyServer
```java
public class NettyServer {  
    public static void main(String[] args) {  
        EventLoopGroup bossGroup = new NioEventLoopGroup();  
        EventLoopGroup workerGroup = new NioEventLoopGroup();  
        try {  
            ChannelFuture bind = new ServerBootstrap()  
                    .group(bossGroup, workerGroup)  
                    .channel(NioServerSocketChannel.class)  
                    .option(ChannelOption.SO_BACKLOG, 128)  
                    .childHandler(new MyChannelInitializer())  
                    .bind(8080);  
            ChannelFuture channelFuture = bind.sync();  
            channelFuture.channel().closeFuture().sync();  
        } catch (Exception e) {  
            System.out.println(e.getMessage());  
        } finally {  
            bossGroup.shutdownGracefully();  
            workerGroup.shutdownGracefully();  
        }  
    }  
}
```

>[!info]
```java
public class NettyClient {  
  
    public static void main(String[] args) throws InterruptedException {  
        Channel channel = new Bootstrap()  
                .group(new NioEventLoopGroup())  
                .channel(NioSocketChannel.class)  
                .handler(new ChannelInitializer<NioSocketChannel>() {  
                    @Override  
                    protected void initChannel(NioSocketChannel ch) {  
                        ch.pipeline().addLast(new StringDecoder(Charset.forName("GBK")));  
                        ch.pipeline().addLast(new StringEncoder(Charset.forName("GBK")));  
                        ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {  
                            @Override  
                            public void channelRead(ChannelHandlerContext ctx, Object msg) {  
                                System.out.println(msg);  
                            }  
                        });  
                    }  
                })  
                .connect("localhost", 8080)  
                .sync()  
                .channel();  
  
        // Get user input  
        new Thread(() -> {  
            Scanner scanner = new Scanner(System.in);  
            String message;  
            while (!"quit".equals(message = scanner.nextLine())) {  
                channel.writeAndFlush(message + "\n");  
            }  
        }).start();  
    }  
  
}
```

群发消息具体体现在这些位置：
在 Channel 建立后，将  Channel 添加到用户组中
```java
public void channelActive(ChannelHandlerContext ctx) throws Exception {  
        // Add to channelGroup  
        ChannelHandler.channelGroup.add(ctx.channel());  
		// 。。。。。。
    }
```

收到消息后，使用 ChannelGroup 来进行消息的群发
```java
    @Override  
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {  
        System.out.println("[Log]: Message = " + msg);  
//        ctx.writeAndFlush("Message received successfully");  
        // Broadcast to all clients        
        ChannelHandler.channelGroup.writeAndFlush(msg);  
        super.channelRead(ctx, msg);  
    }  
```

最后，注意断开的时候要将用户从 ChannelGroup 中删除：
```java
    @Override  
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {  
        System.out.println("[Log]: Client disconnected");  
        ChannelHandler.channelGroup.remove(ctx.channel());  
        super.channelInactive(ctx);  
    }  
```

# Netty Client
>[!info] Netty Client
```java
public class NettyClient {  
    public static void main(String[] args) throws InterruptedException {  
        Channel channel = new Bootstrap()  
                .group(new NioEventLoopGroup())  
                .channel(NioSocketChannel.class)  
                .handler(new ChannelInitializer<NioSocketChannel>() {  
                    @Override  
                    protected void initChannel(NioSocketChannel ch) {  
                        System.out.println("[Log]: Connected to server, channel id is " + ch.id());  
                        ch.pipeline().addLast(new StringDecoder(Charset.forName("GBK")));  
                        ch.pipeline().addLast(new StringEncoder(Charset.forName("GBK")));  
                        ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {  
                            @Override  
                            public void channelRead(ChannelHandlerContext ctx, Object msg) {  
                                System.out.println(msg);  
                            }  
                        });  
                    }  
                })  
                .connect("localhost", 8080)  
                .sync()  
                .channel();  
        // Get user input  
        new Thread(() -> {  
            Scanner scanner = new Scanner(System.in);  
            String message;  
            while (!"quit".equals(message = scanner.nextLine())) {  
                channel.writeAndFlush(message + "\n");  
            }  
        }).start();  
    }  
}
```

# 自定义编码解码器，处理黏包半包问题
在实际应用场景里，只要是支持sokcet通信的都可以和Netty交互，比如中继器、下位机、PLC等。
这些场景下就非常需要自定义编码解码器，来处理字节码传输，并控制半包、粘包以及安全问题。那么本章节我们通过实现ByteToMessageDecoder、MessageToByteEncoder来实现我们的需求。
格式：
```
0x02 message length {...............} 0x03
```

>[!info] MyDecoder
```java
public class MyDecoder extends ByteToMessageDecoder {  
    private static final byte START_FLAG = 0x02;  
    private static final byte END_FLAG = 0x03;  
    private static final int BASE_LENGTH = 4;  
    private static final int MESSAGE_LENGTH_INDEX = 1;  
    /**  
     * Decodes the specified buffer into [one or more messages] and adds each decoded message to the out     * @param ctx the {@link ChannelHandlerContext} which this {@link ByteToMessageDecoder} belongs to  
     * @param in the {@link ByteBuf} from which to read data  
     * @param out the {@link List} to which decoded messages should be added  
     */    @Override  
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {  
        // Readable length do not retch base length.  
        if (in.readableBytes() < BASE_LENGTH) {  
            return;  
        }  
        // Record bag header.  
        int beginIdx;  
        while (true) {  
            beginIdx = in.readerIndex();  
            in.markReaderIndex();  
            // The start mark.  
            if (in.readByte() == START_FLAG) break;  
            // If the rest bytes less than base length, we should wait for next delivered data.  
            if (in.readableBytes() < BASE_LENGTH) return;  
        }  
        // Message with bag header, but no data.  
        if (in.readableBytes() <= 1) {  
            return;  
        }  
        // Get message length.  
        ByteBuf byteBuf = in.readBytes(MESSAGE_LENGTH_INDEX);  
        int msgLength = byteBuf.getByte(0);  
  
        ByteBuf msgContent = in.readBytes(msgLength);  
        byte endByte = in.readByte();  
        if (endByte != END_FLAG) {  
            in.readerIndex(beginIdx);  
            return;  
        }  
        out.add(msgContent.toString(Charset.forName("GBK")));  
    }  
}
```

>[!info] MyEncoder
```java
public class MyEncoder extends MessageToByteEncoder<Object> {  
    @Override  
    protected void encode(ChannelHandlerContext ctx, Object msg, ByteBuf out) throws Exception {  
        String msgString = msg.toString();  
        ByteBuffer encode = Charset.forName("GBK").encode(msgString);  
        byte[] bytes = encode.array();  
        int msgLength = encode.limit();  
        if (msgLength > Byte.MAX_VALUE) {  
            throw new RuntimeException("msg too long");  
        }  
  
        byte[] send = new byte[msgLength + 3];  
        System.arraycopy(bytes, 0, send, 2, msgLength);  
  
        send[0] = 0x02;  
        send[1] = (byte) msgLength;  
        send[send.length - 1] = 0x03;  
        out.writeBytes(send);  
    }  
}
```
# 关于 @Shared 注解
`ChannelInboundHandler` 拦截和处理入站事件，`ChannelOutboundHandler` 拦截和处理出站事件。
`ChannelHandler` 和 `ChannelHandlerContext` 通过组合或继承的方式关联到一起成对使用。
事件通过 `ChannelHandlerContext` 主动调用如 read(msg)、write(msg) 和 fireXXX() 等方法，将事件传播到下一个处理器。
注意：入站事件在 `ChannelPipeline` 双向链表中由头到尾正向传播，出站事件则方向相反。 当客户端连接到服务器时，`Netty` 新建一个 `ChannelPipeline` 处理其中的事件，而一个 `ChannelPipeline` 中含有若干 `ChannelHandler`。
如果每个客户端连接都新建一个 `ChannelHandler` 实例，当有大量客户端时，服务器将保存大量的 `ChannelHandler` 实例。为此，Netty 提供了 `Sharable` 注解，如果一个 `ChannelHandler` 与状态无关，那么可将其标注为 `Sharable`，如此，服务器只需保存一个实例就能处理所有客户端的事件。
比如 StringEncoder 上就存在这个注解：
```java
@Sharable  
public class StringEncoder extends MessageToMessageEncoder<CharSequence>
```

# Netty UDP 通信方式案例 Demo
在Netty通信中UDP的实现方式也非常简单，只要注意部分代码区别于TCP即可。本章节需要注意的知识点 ；NioDatagramChannel、ChannelOption.SO_BROADCAST
Internet 协议集支持一个无连接的传输协议，该协议称为用户数据报协议（UDP，User Datagram Protocol）。
UDP 为应用程序提供了一种无需建立连接就可以发送封装的 IP 数据报的方法。RFC 768 描述了 UDP。 Internet 的传输层有两个主要协议，互为补充。
无连接的是 UDP，它除了给应用程序发送数据包功能并允许它们在所需的层次上架构自己的协议之外，几乎没有做什么特别的的事情。面向连接的是 TCP，该协议几乎做了所有的事情。
>[!info] NettyClient
```java
public class NettyClient {  
    public static void main(String[] args) {  
        NioEventLoopGroup eventExecutors = new NioEventLoopGroup();  
        try {  
            Bootstrap handler = new Bootstrap()  
                    .group(eventExecutors)  
                    .channel(NioDatagramChannel.class)  
                    .handler(new MyChannelInitializer());  
            Channel channel = handler.bind(8081).sync().channel();  
            channel.writeAndFlush(new DatagramPacket(Unpooled.copiedBuffer("Hello", Charset.forName("GBK")),  
                    new InetSocketAddress("localhost", 8080))).sync();  
            channel.closeFuture().sync();  
        } catch (InterruptedException e) {  
            throw new RuntimeException(e);  
        }  
    }  
}
```

>[!info] MyChannelInitializer
```java
public class MyChannelInitializer extends ChannelInitializer<NioDatagramChannel> {  
    @Override  
    protected void initChannel(NioDatagramChannel ch) throws Exception {  
        ChannelPipeline pipeline = ch.pipeline();  
        // 解码转String，注意调整自己的编码格式GBK、UTF-8  
        //pipeline.addLast("stringDecoder", new StringDecoder(Charset.forName("GBK")));        pipeline.addLast(new MyClientHandler());  
    }  
}
```

>[!info] MyClientHandler
```java
public class MyClientHandler extends SimpleChannelInboundHandler<DatagramPacket> {  
    @Override  
    protected void channelRead0(ChannelHandlerContext ctx, DatagramPacket msg) throws Exception {  
        String message = msg.content().toString(Charset.forName("GBK"));  
        System.out.println("[Log]: " + new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()) +  
                " UDP client get message " + message);  
    }  
}
```

>[!info] NettyServer
```java
public class NettyServer {  
    public static void main(String[] args) {  
        EventLoopGroup group = new NioEventLoopGroup();  
        try {  
            Bootstrap b = new Bootstrap();  
            b.group(group)  
                    .channel(NioDatagramChannel.class)  
                    .option(ChannelOption.SO_BROADCAST, true)  
                    .option(ChannelOption.SO_RCVBUF, 2048 * 1024)  
                    .option(ChannelOption.SO_SNDBUF, 1024 * 1024)  
                    .handler(new MyChannelInitializer());  
            ChannelFuture f = b.bind(8080).sync();  
            f.channel().closeFuture().sync();  
        } catch (InterruptedException e) {  
            throw new RuntimeException(e);  
        } finally {  
            // Close server gracefully
            group.shutdownGracefully();  
        }  
    }  
}
```

>[!info] MyChannelInitializer
```java
public class MyChannelInitializer extends ChannelInitializer<NioDatagramChannel> {  
    private final EventLoopGroup group = new NioEventLoopGroup();  
    @Override  
    protected void initChannel(NioDatagramChannel ch) {  
        ChannelPipeline pipeline = ch.pipeline();  
        pipeline.addLast(group, new MyServerHandler());  
    }  
}
```

>[!info] MyServerHandler
```java
public class MyServerHandler extends SimpleChannelInboundHandler<DatagramPacket> {  
    @Override  
    protected void channelRead0(ChannelHandlerContext ctx, DatagramPacket packet) throws Exception {  
        String msg = packet.content().toString(Charset.forName("GBK"));  
        System.out.println("[Log]: " + new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()) +  
                " UDP server get message " + msg);  
  
        // Send message to client  
        String json = "Hello, your message have been received!";  
        byte[] bytes = json.getBytes(Charset.forName("GBK"));  
        DatagramPacket data = new DatagramPacket(Unpooled.copiedBuffer(bytes), packet.sender());  
        ctx.writeAndFlush(data);  
    }  
}
```

# 简单实现一个基于 Netty 的 HTTP 服务
Netty不仅可以搭建Socket服务，也可以搭建Http、Https服务。
超文本传输协议（HTTP，HyperText Transfer Protocol)是互联网上应用最为广泛的一种网络协议。

> 在后端开发中接触HTTP协议的比较多，目前大部分都是基于Servlet容器实现的Http服务，往往有一些核心子系统对性能的要求非常高，这个时候我们可以考虑采用NIO的网络模型来实现HTTP服务，以此提高性能和吞吐量，Netty除了开发网络应用非常方便，还内置了HTTP相关的编解码器，让用户可以很方便的开发出高性能的HTTP协议的服务，Spring Webflux默认是使用的Netty。

>[!info] NettyServer
```java
public class NettyServer {  
    public static void main(String[] args) {  
        EventLoopGroup parentGroup = new NioEventLoopGroup();  
        EventLoopGroup childGroup = new NioEventLoopGroup();  
        try {  
            ServerBootstrap b = new ServerBootstrap();  
            b.group(parentGroup, childGroup)  
                    .channel(NioServerSocketChannel.class)  
                    .option(ChannelOption.SO_BACKLOG, 128)  
                    .childOption(ChannelOption.SO_KEEPALIVE, true)  
                    .childHandler(new MyChannelInitializer());  
            ChannelFuture f = b.bind(7777).sync();  
            f.channel().closeFuture().sync();  
        } catch (InterruptedException e) {  
            System.out.println(e.getMessage());  
        } finally {  
            childGroup.shutdownGracefully();  
            parentGroup.shutdownGracefully();  
        }  
    }  
}
```

>[!info] MyChannelInitializer 将 Netty 提供的 HTTP 编解码器添加到 Pipeline 中
```java
public class MyChannelInitializer extends ChannelInitializer<NioSocketChannel> {  
    @Override  
    protected void initChannel(**NioSocketChannel** ch) {  
        ch.pipeline().addLast(new HttpResponseEncoder())  
                .addLast(new HttpRequestDecoder())  
                .addLast(new HttpResponseDecoder())  
                .addLast(new MyServerHandler());  
    }  
}
```

>[!info] MyServerHandler
```java
public class MyServerHandler extends ChannelInboundHandlerAdapter {  
  
    private void sendResponse(RandomAccessFile raf, ChannelHandlerContext ctx, Object msg, String contentType) throws IOException {  
        long fileLength = raf.length();  
        HttpResponse response = new DefaultHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.OK);  
        HttpUtil.setContentLength(response, fileLength);  
        response.headers().set(HttpHeaderNames.CONTENT_TYPE, contentType);  
  
        ctx.write(response);  
  
        ChannelFuture sendFileFuture = ctx.write(new DefaultFileRegion(raf.getChannel(), 0, fileLength), ctx.newProgressivePromise());  
        sendFileFuture.addListener(new ChannelProgressiveFutureListener() {  
            @Override  
            public void operationProgressed(ChannelProgressiveFuture future, long progress, long total) {  
                if (total < 0) { // total unknown  
                    System.err.println("Transfer progress: " + progress);  
                } else {  
                    System.err.println("Transfer progress: " + progress + " / " + total);  
                }  
            }  
            @Override  
            public void operationComplete(ChannelProgressiveFuture future) {  
                System.err.println("Transfer complete.");  
            }  
        });  
  
        ChannelFuture lastContentFuture = ctx.writeAndFlush(LastHttpContent.EMPTY_LAST_CONTENT);  
  
        // Close the connection if client does not want to continue, after writing last content.  
        if (msg instanceof HttpRequest && !HttpUtil.isKeepAlive((HttpRequest) msg)) {  
            lastContentFuture.addListener(ChannelFutureListener.CLOSE);  
        }  
    }  
  
    @Override  
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws IOException {  
        String contentType = "text/html";  
        Path path = Paths.get("ryan-demo-netty-http/src/main/resources/Hello.html");  
        try {  
            if (msg instanceof HttpRequest) {  
                HttpRequest request = (HttpRequest) msg;  
                System.out.println("URI" + request.uri());  
                if (request.uri().equals("/img.jpg")) {  
                    contentType = "image/jpeg";  
                    path = Paths.get("ryan-demo-netty-http/src/main/resources/img.jpg");  
                } else if (request.uri().equals("/laoba.jpg")) {  
                    contentType = "image/jpeg";  
                    path = Paths.get("ryan-demo-netty-http/src/main/resources/laoba.jpg");  
                }  
                System.out.println(msg);  
            } else if (msg instanceof HttpContent) {  
                LastHttpContent httpContent = (LastHttpContent) msg;  
                ByteBuf byteData = httpContent.content();  
                if (!(byteData instanceof EmptyByteBuf)) {  
                    // 接收msg消息  
                    byte[] msgByte = new byte[byteData.readableBytes()];  
                    byteData.readBytes(msgByte);  
                    System.out.println(new String(msgByte, StandardCharsets.UTF_8));  
                }  
            }  
            File file = path.toFile();  
            RandomAccessFile raf = new RandomAccessFile(file, "r");  
            sendResponse(raf, ctx, msg, contentType);  
        } catch (IOException e) {  
            if (e.getMessage().contains("Connection reset")) {  
                System.err.println("Connection reset by peer: " + e.getMessage());  
            } else {  
                throw e;  
            }  
        }  
    }  
  
    @Override  
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {  
        if (cause instanceof IOException) {  
            // 处理 IOException，例如记录日志  
            System.err.println("IOException occurred: " + cause.getMessage());  
        } else {  
            // 处理其他异常  
            throw new RuntimeException(cause);  
        }  
        // 关闭连接  
        ctx.close();  
    }  
  
}
```