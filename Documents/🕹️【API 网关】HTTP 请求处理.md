-  项目地址：[https://github.com/KkQ36/api-gateway/tree/20241024_gateway_http](https://github.com/KkQ36/api-gateway/tree/20241024_gateway_http)
- 项目分支：`20241024_gateway_http`
# 基本介绍
![[【API 网关】Session 流程.png]]
HTTP请求到 **API网关**，网关再去调用到对应的RPC服务，那么这样一个流程一次请求，可以把它抽象为是做了一次 **会话（Session）** 操作。
1. 之所以称之为会话，是因为一次 HTTP 请求，就要完成；建立连接、协议转换、方法映射、泛化调用、返回结果等一些列操作。而在这些操作过程中的各类行为处理，其实也类似于ORM框架，只不过一个是对数据库的处理，一个是对 RPC服务的处理。
2. 此外之所以要单独创建出一个 api-gateway-core 的工程，是因为我们要把这个工程独立于各种容器，它并不是直接与 SpringBoot 一起开发，因为那样会让组件失去灵活性。它的存在更应该像是一个 ORM 框架，可以独立使用，谁也都可以结合。
# 项目结构
```
.
├── api-gateway-core
│   ├── pom.xml
│   └── src
│       ├── main
│       │   ├── java
│       │   │   └── com
│       │   │       └── ryan
│       │   │           └── core
│       │   │               └── session
│       │   │                   ├── BaseHandler.java
│       │   │                   ├── SessionChannelInitializer.java
│       │   │                   ├── SessionServer.java
│       │   │                   └── handlers
│       │   │                       └── SessionServerHandler.java
│       │   └── resources
│       └── test
│           ├── java
│           │   └── com
│           │       └── ryan
│           │           └── core
│           │               └── ApiTest.java
│           └── resources
└── pom.xml
```
# 核心代码
本部分属于基本的 Netty Server 的创建流程：
>[!info] com.ryan.core.session.SessionServer
```java
public class SessionServer implements Callable<Channel> {  
    private final Logger logger = LoggerFactory.getLogger(SessionServer.class);  
    private final EventLoopGroup bossGroup = new NioEventLoopGroup();  
    private final EventLoopGroup workerGroup = new NioEventLoopGroup();  
    private Channel channel;  
    @Override  
    public Channel call() {  
        ChannelFuture channelFuture = null;  
        try {  
            ServerBootstrap serverBootstrap = new ServerBootstrap();  
            serverBootstrap.group(bossGroup, workerGroup)  
                    .channel(NioServerSocketChannel.class)  
                    .childHandler(new SessionChannelInitializer());  
  
            channelFuture = serverBootstrap.bind(new InetSocketAddress(8080)).syncUninterruptibly();  
            this.channel = channelFuture.channel();  
        } catch (Exception e) {  
            logger.error("Server start error.", e);  
        } finally {  
            if (null != channelFuture && channelFuture.isSuccess()) {  
                logger.info("Server start success.");  
            } else {  
                logger.error("Server start error.");  
            }  
        }  
        return channel;  
    }  
}
```

>[!info] com.ryan.core.session.SessionChannelInitializer
```java
public class SessionChannelInitializer extends ChannelInitializer<NioSocketChannel> {  
    @Override  
    protected void initChannel(NioSocketChannel channel) {  
        ChannelPipeline line = channel.pipeline();  
        line.addLast(new HttpRequestDecoder());  
        line.addLast(new HttpResponseEncoder());  
        line.addLast(new HttpObjectAggregator(1024 * 1024));  
        line.addLast(new SessionServerHandler());  
    }  
}
```

>[!info] com.ryan.core.session.BaseHandler
```java
public abstract class BaseHandler<T> extends SimpleChannelInboundHandler<T> {  
    @Override  
    protected void channelRead0(ChannelHandlerContext ctx, T msg) throws Exception {  
        session(ctx, ctx.channel(), msg);  
    }  
    protected abstract void session(ChannelHandlerContext ctx, final Channel channel, T request);  
}
```

>[!info] com.ryan.core.session.handlers.SessionServerHandler
```java
public class SessionServerHandler extends BaseHandler<FullHttpRequest> {  
    private final Logger logger = LoggerFactory.getLogger(SessionServerHandler.class);  
    @Override  
    protected void session(ChannelHandlerContext ctx, Channel channel, FullHttpRequest request) {  
        logger.info("GateWay get request uri：{} method：{}", request.uri(), request.method());  
        DefaultFullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.OK);  
        response.content().writeBytes(JSON.toJSONBytes("The URI you assessed has been control by Gateway URI：" +  
                request.uri(), SerializerFeature.PrettyFormat));  
        // Config response headers  
        HttpHeaders heads = response.headers();  
        heads.add(HttpHeaderNames.CONTENT_TYPE, HttpHeaderValues.APPLICATION_JSON + "; charset=UTF-8");  
        heads.add(HttpHeaderNames.CONTENT_LENGTH, response.content().readableBytes());  
        heads.add(HttpHeaderNames.CONNECTION, HttpHeaderValues.KEEP_ALIVE);  
        // Config CORS  
        heads.add(HttpHeaderNames.ACCESS_CONTROL_ALLOW_ORIGIN, "*");  
        heads.add(HttpHeaderNames.ACCESS_CONTROL_ALLOW_HEADERS, "*");  
        heads.add(HttpHeaderNames.ACCESS_CONTROL_ALLOW_METHODS, "GET, POST, PUT, DELETE");  
        heads.add(HttpHeaderNames.ACCESS_CONTROL_ALLOW_CREDENTIALS, "true");  
        channel.writeAndFlush(response);  
    }  
}
```

