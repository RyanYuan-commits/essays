# SpringBoot 整合 Netty
在实际的开发中，我们需要对netty服务进行更多的操作，包括；获取它的状态信息、启动/停止、对客户端用户强制下线等等，为此我们需要把netty服务加入到web系统中，那么本章节介绍如何将Netty与SpringBoot整合。
环境：SpringBoot 3.34、Netty 4.1.39.Final

>[!info] pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>  
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">  
    <modelVersion>4.0.0</modelVersion>  
    <parent>  
       <groupId>org.springframework.boot</groupId>  
       <artifactId>spring-boot-starter-parent</artifactId>  
       <version>3.3.4</version>  
       <relativePath/> <!-- lookup parent from repository -->  
    </parent>  
    <groupId>com.ryan</groupId>  
    <artifactId>ryan-demo-netty-2-1</artifactId>  
    <version>0.0.1-SNAPSHOT</version>  
    <name>ryan-demo-netty-2-1</name>  
    <description>ryan-demo-netty-2-1</description>  
    <properties>  
       <java.version>17</java.version>  
    </properties>  
    <dependencies>  
       <dependency>  
          <groupId>io.netty</groupId>  
          <artifactId>netty-all</artifactId>  
       </dependency>  
       <dependency>  
          <groupId>org.springframework.boot</groupId>  
          <artifactId>spring-boot-starter-web</artifactId>  
       </dependency>  
       <dependency>  
          <groupId>org.springframework.boot</groupId>  
          <artifactId>spring-boot-devtools</artifactId>  
          <scope>runtime</scope>  
          <optional>true</optional>  
       </dependency>  
       <dependency>  
          <groupId>org.projectlombok</groupId>  
          <artifactId>lombok</artifactId>  
          <optional>true</optional>  
       </dependency>  
       <dependency>  
          <groupId>org.springframework.boot</groupId>  
          <artifactId>spring-boot-starter-test</artifactId>  
          <scope>test</scope>  
       </dependency>  
       <dependency>  
          <groupId>io.projectreactor</groupId>  
          <artifactId>reactor-test</artifactId>  
          <scope>test</scope>  
       </dependency>  
    </dependencies>  
    <build>  
       <plugins>  
          <plugin>  
             <groupId>org.springframework.boot</groupId>  
             <artifactId>spring-boot-maven-plugin</artifactId>  
             <configuration>  
                <excludes>  
                   <exclude>  
                      <groupId>org.projectlombok</groupId>  
                      <artifactId>lombok</artifactId>  
                   </exclude>  
                </excludes>  
             </configuration>  
          </plugin>  
       </plugins>  
    </build>  
</project>
```

>[!info] MyNettyServer.java
```java
@Getter  
@Component("nettyServer")  
public class MyNettyServer {  
    //配置服务端NIO线程组  
    private final EventLoopGroup parentGroup = new NioEventLoopGroup();  
    private final EventLoopGroup childGroup = new NioEventLoopGroup();  
    private Channel channel;  
  
    public ChannelFuture bing(InetSocketAddress address) {  
        ChannelFuture channelFuture = null;  
        try {  
            ServerBootstrap b = new ServerBootstrap();  
            b.group(parentGroup, childGroup)  
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 128)  
                    .childHandler(new MyChannelInitializer());  
  
            channelFuture = b.bind(address).syncUninterruptibly();  
            channel = channelFuture.channel();  
        } catch (Exception e) {  
            System.out.println(e.getMessage());  
        }  
        return channelFuture;  
    }  
  
    public void destroy() {  
        if (null == channel) return;  
        channel.close();  
        parentGroup.shutdownGracefully();  
        childGroup.shutdownGracefully();  
    }  
}
```

>[!info] MyChannelInitializer
```java
public class MyChannelInitializer extends ChannelInitializer<NioSocketChannel> {  
    @Override  
    protected void initChannel(NioSocketChannel channel) throws Exception {  
        channel.pipeline().addLast(new LineBasedFrameDecoder(1024));  
        channel.pipeline().addLast(new StringDecoder(Charset.forName("GBK")));  
        channel.pipeline().addLast(new StringEncoder(Charset.forName("GBK")));  
        channel.pipeline().addLast(new MyServerHandler());  
    }  
}
```

>[!info] MyServerHandler
```java
public class MyServerHandler extends ChannelInboundHandlerAdapter {  
    @Override  
    public void channelActive(ChannelHandlerContext ctx) throws Exception {  
        SocketChannel channel = (SocketChannel) ctx.channel();  
        System.out.println("[Log]: ip = " + channel.socket().getPort());  
        System.out.println("[Log]: port = " + channel.socket().getPort());  
        String str = "[" + " " + new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()) + "]" +  
                "Connected successfully!";  
        ctx.writeAndFlush(str);  
    }  
      
    @Override  
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {  
        System.out.println("[Log]: " + "channelInactive");  
    }  
  
    @Override  
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {  
        String str = "[" + new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()) + "]: "  
                + "Server got message: " + msg;  
        ctx.writeAndFlush(str);  
    }  
      
    @Override  
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {  
        ctx.close();  
        System.out.println("[Log]: " + cause.getMessage());  
    }  
  
}
```

>[!info] NettyController
```java
@RestController  
@RequestMapping("/api")  
public class NettyController {  
    @Resource(name = "nettyServer")  
    MyNettyServer myNettyServer;  
  
    @GetMapping("/localAddress")  
    public String localAddress(){  
        return "Netty Server local address is " + myNettyServer.getChannel().localAddress();  
    }  
  
    @GetMapping("isOpen")  
    public boolean isOpen(){  
        return myNettyServer.getChannel().isOpen();  
    }  
}
```

# 使用 ProtoBuf 传输数据
## ProtoBuf 简介
![[protobuf icon.png#pic_center]]
[Protocol Buffer](https://protobuf.dev/)
[Protocol Buffer GitHub](https://github.com/protocolbuffers/protobuf)
Protocol Buffers（简称 ProtoBuf）是由 Google 开发的一种用于序列化结构化数据的语言和平台无关的机制。它类似于 XML 或 JSON，但更小、更快、更简单。
1. **高效性**：ProtoBuf 使用二进制格式进行数据序列化，比文本格式（如 XML 或 JSON）更紧凑，解析速度更快。
2. **跨语言支持**：ProtoBuf 支持多种编程语言，包括 Java、C++、Python、Go 等，使得在不同语言的系统之间传输数据变得简单。
3. **向后兼容性**：ProtoBuf 允许在不破坏现有数据格式的情况下对数据结构进行演变。你可以添加新的字段或弃用旧的字段，而不会影响旧的代码。
4. **简单的定义语法**：使用 `.proto` 文件定义数据结构，语法简单明了。编译器会根据这些定义生成相应语言的代码。
5. **广泛应用**：ProtoBuf 常用于 RPC 系统（如 gRPC）和数据存储系统中，尤其是在需要高性能和高效数据传输的场景下。
一个简单的 ProtoBuf 示例：
```protobuf
syntax = "proto3";

message Person {
  string name = 1;
  int32 id = 2;
  string email = 3;
}
```
在这个示例中，定义了一个 `Person` 消息类型，包含三个字段：`name`、`id` 和 `email`。每个字段都有一个唯一的编号，用于在二进制格式中标识字段。
## QuickStart
This tutorial provides a basic Java programmer’s introduction to working with protocol buffers. By walking through creating a simple example application, it shows you how to
- Define message formats in a `.proto` file.
- Use the protocol buffer compiler.
- Use the Java protocol buffer API to write and read messages.
上面给出的是官网推荐的 Quick Start 步骤，首先来定义一个 .proto 后缀的文件。
```protobuf
syntax = "proto3";  
  
package com.ryan.domain;  
  
option java_package = "com.ryan.domain";  
option java_multiple_files = true;  
option java_outer_classname = "MsgInfo";  
  
message MsgBody {  
  
  string channelId = 1;  
  string msgInfo = 2;  
  
}
```

然后去官网下载 proto 编译器：
![[proto compiler download.png]]
选择合适的版本安装即可，安装成功后配置环境变量，我们就可以使用 `protoc` 来进行文件的编译了。
```bash
protoc -I=$SRC_DIR --java_out=$DST_DIR $SRC_DIR/addressbook.proto
```
然后我们就可以在指定的文件夹中得到这些内容
```java
-->
| MsgBody
| MsgBodyOrBuilder
| MsgInfo
```

此时就可以使用 Netty 提供的 ProtoBuf 编解码器，对我们创建的 Proto 结构进行序列化、传输等操作了：
>[!info] NettyClient
```java
public class NettyClient {  
    public static void main(String[] args) {  
        new NettyClient().connect("127.0.0.1", 7397);  
    }  
    private void connect(String inetHost, int inetPort) {  
        EventLoopGroup workerGroup = new NioEventLoopGroup();  
        try {  
            Bootstrap b = new Bootstrap();  
            b.group(workerGroup);  
            b.channel(NioSocketChannel.class);  
            b.option(ChannelOption.AUTO_READ, true);  
            b.handler(new MyChannelInitializer());  
            ChannelFuture f = b.connect(inetHost, inetPort).sync();  
  
            f.channel().writeAndFlush(MsgUtil.buildMsg(f.channel().id().toString(),"hello protobuf"));  
            f.channel().closeFuture().sync();  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        } finally {  
            workerGroup.shutdownGracefully();  
        }  
    }  
}
```

>[!info] MyChannelInitializer
```java
public class MyChannelInitializer extends ChannelInitializer<NioSocketChannel> {  
	@Override  
	protected void initChannel(NioSocketChannel channel) {  
	    channel.pipeline().addLast(new ProtobufVarint32FrameDecoder());  
	    channel.pipeline().addLast(new ProtobufDecoder(MsgBody.getDefaultInstance()));  
	    channel.pipeline().addLast(new ProtobufVarint32LengthFieldPrepender());  
	    channel.pipeline().addLast(new ProtobufEncoder());  
	    channel.pipeline().addLast(new MyClientHandler());  
	} 
}
```

>[!info] MyClientHandler
```java
public class MyClientHandler extends ChannelInboundHandlerAdapter {  
    @Override  
    public void channelRead(ChannelHandlerContext ctx, Object msg) {  
        System.out.println(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()) + "Client got message class: " + msg.getClass());  
        System.out.println(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()) + "Client got message: " + msg);  
    }  
}
```
# Netty 传输 Java 对象
Netty在实际应用级开发中，有时候某些特定场景下会需要使用Java对象类型进行传输，但是如果使用Java本身序列化进行传输，那么对性能的损耗比较大。为此我们需要借助protostuff-core的工具包将对象以二进制形式传输并做编码解码处理。与直接使用protobuf二进制传输方式不同，这里不需要定义proto文件，而是需要实现对象类型编码解码器，用以传输自定义Java对象。
> protostuff 基于Google protobuf，但是提供了更多的功能和更简易的用法。其中，protostuff-runtime 实现了无需预编译对 java bean 进行 protobuf 序列化/反序列化的能力。protostuff-runtime 的局限是序列化前需预先传入 schema，反序列化不负责对象的创建只负责复制，因而必须提供默认构造函数。此外，protostuff 还可以按照protobuf的配置序列化成 json/yaml/xml 等格式。在性能上，protostuff不输原生的protobuf，甚至有反超之势。
1. 支持protostuff-compiler产生的消息
2. 支持现有的POJO对象
3. 支持现有的protoc产生的Java消息
4. 与各种移动平台的互操作能力（Android、Kindle、j2me）
5. 支持转码

使用 Netty 来传输 Java 对象的话，重要的就是进行编码和解码
首先要规定一个传输的对象类型，之后传输的数据都将通过这个 Java 对象进行传输：

>[!info] MsgInfo
```java
public class MsgInfo {  
    private String channelId;  
    private String msgContent;  
    public MsgInfo() {  
    } 
    public MsgInfo(String channelId, String msgContent) {  
        this.channelId = channelId;  
        this.msgContent = msgContent;  
    }  
    public String getChannelId() {  
        return channelId;  
    }  
    public void setChannelId(String channelId) {  
        this.channelId = channelId;  
    }  
    public String getMsgContent() {  
        return msgContent;  
    }  
    public void setMsgContent(String msgContent) {  
        this.msgContent = msgContent;  
    }  
}
```

>[!info] pom.xml：引入 protostuff 包来进行 Java 对象的序列化和反序列化
```xml
<?xml version="1.0" encoding="UTF-8"?>  
<project xmlns="http://maven.apache.org/POM/4.0.0"  
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">  
    <parent>  
        <artifactId>itstack-demo-netty</artifactId>  
        <groupId>org.itatack.demo</groupId>  
        <version>1.0.0-SNAPSHOT</version>  
    </parent>  
    <modelVersion>4.0.0</modelVersion>  
    <artifactId>itstack-demo-netty-2-03</artifactId>  
    <properties>  
        <protostuff.version>1.0.10</protostuff.version>  
        <objenesis.version>2.4</objenesis.version>  
    </properties>  
    <dependencies>  
        <!-- Protostuff -->  
        <dependency>  
            <groupId>com.dyuproject.protostuff</groupId>  
            <artifactId>protostuff-core</artifactId>  
            <version>${protostuff.version}</version>  
        </dependency>  
        <dependency>  
            <groupId>com.dyuproject.protostuff</groupId>  
            <artifactId>protostuff-runtime</artifactId>  
            <version>${protostuff.version}</version>  
        </dependency>  
        <dependency>  
            <groupId>org.objenesis</groupId>  
            <artifactId>objenesis</artifactId>  
            <version>${objenesis.version}</version>  
        </dependency>  
        <!-- fastjson -->  
        <dependency>  
            <groupId>com.alibaba</groupId>  
            <artifactId>fastjson</artifactId>  
            <version>1.2.58</version>  
        </dependency>  
    </dependencies>  
</project>
```

在编解码的过程中，我们规定文件的第一个字节是这条消息的大小
>[!info] ObjDecoder：对象解码器
```java
public class ObjDecoder extends ByteToMessageDecoder {  
    @Override  
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {  
        if (in.readableBytes() < 4) {  
            return;  
        }  
        in.markReaderIndex();  
        int dataLength = in.readInt();  
        if (in.readableBytes() < dataLength) {  
            in.resetReaderIndex();  
            return;  
        }  
        byte[] data = new byte[dataLength];  
        in.readBytes(data);  
        MsgInfo deserialize = SerializationUtil.deserialize(data, MsgInfo.class);  
        out.add(deserialize);  
    }  
}
```

>[!info] 对象编码器
```java
public class ObjEncoder extends MessageToByteEncoder<Object> {  
    @Override  
    protected void encode(ChannelHandlerContext ctx, Object in, ByteBuf out)  {  
        if (in instanceof MsgInfo) {  
            byte[] data = SerializationUtil.serialize(in);  
            out.writeInt(data.length);  
            out.writeBytes(data);  
        }  
    }  
}
```
# Netty 发送文件、分片发送、断点续传
1. 在实际应用中我们经常使用到网盘服务，他们可以高效的上传下载较大文件。那么这些高性能文件传输服务，都需要实现的分片发送、断点续传功能。
2. 在 `Java` 文件操作中有 `RandomAccessFile` 类，他可以支持文件的定位读取和写入，这样就满足了我们对文件分片的最基础需求。
3. Netty服务端启动后，可以向客户端发送文件传输指令；允许接收文件、控制读取位点、记录传输标记、文件接收完成。
4. 为了保证传输性能我们采用 `protostuff` 二进制流进行传输。
5. 读取文件的时候需要注意，我们设定`byte[1024]`为默认读取范围，但当读取到最后的时候可能不足1024个字节，就会出现空字节。
   这个时候需要去掉空字节，否则我们的文件写入会多额外信息，导致文件不能打开｛zip、war、exe、jar等｝。
