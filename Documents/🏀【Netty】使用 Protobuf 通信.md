---
created: 2025-09-27 22:34:06
updated: 2025-09-27 22:34:06
---
Protobuf 全称是 Google Protocol Buffer，是 Google 提出的一种数据交换的格式，是一套类似 JSON 或者 XML 的数据传输格式和规范，用于不同应用或进程之间进行通信。
Protobuf 具有以下特点：
	（1）语言无关，平台无关 Protobuf 支持 Java、 C++,、Python、JavaScript 等多种语言，支持跨多个平台。
	（2）高效，比XML更小（3~10倍），更快（20 ~ 100倍），更为简单。
	（3）扩展性，兼容性好可以更新数据结构，而不影响和破坏原有的旧程序。
==Protobuf 既独立于语言，又独立于平台==。Google 官方提供了多种语言的实现：Java、C#、C++、GO、JavaScript和Python。
Protobuf 的编码过程为：使用预先定义的 Message 数据结构将实际的传输数据进行打包，然后编码成二进制的码流进行传输或者存储。Protobuf 的解码过程则刚好与编码过程相反：将二进制码流解码成 Protobuf 自己定义的 Message 结构的 POJO 实例。
与 JSON、XML 相比，Protobuf 算是后起之秀，只是 Protobuf 更加适合于高性能、快速响应的数据传输应用场景。Protobuf 数据包是一种二进制的格式，相对于文本格式的数据交换（JSON、XML）来说，速度要快很多。由于 Protobuf 优异的性能，使得它更加适用于分布式应用场景下的数据通信或者异构环境下的数据交换。另外，JSON、XML 是文本格式，数据具有可读性；而 Protobuf 是二进制数据格式，数据本身不具有可读性，只有反序列化之后才能得到真正可读的数据。
正因为 Protobuf 是二进制数据格式，数据序列化之后，体积相比 JSON 和 XML 要小，更加适合网络传输。
总体来说，在一个需要大量数据传输的应用场景中，因为数据量很大，那么选择 Protobuf 可以明显地减少传输的数据量和提升网络 IO 的速度。对于打造一款高性能的通信服务器来说，Protobuf 传输协议是最高性能的传输协议之一。微信的消息传输就采用了 Protobuf 协议。
# 一个简单的 proto 文件案例
Protobuf 使用 proto 文件来预先定义消息的格式，数据包是按照 proto 文件定义的消息格式完成二进制码流的编码和解码。
proto 文件简单来说就是一个消息的协议文件，这个文件的后缀为 ".proto"。
```protobuf
// [开始头部声明]  
syntax = "proto3";  
package com.ryan.protobuf;  
// [结束头部声明]  
  
// [开始 java 选项配置]  
option java_package = "com.ryan.protobuf.pojo";  
option java_outer_classname = "MsgProto";  
// [结束 java 选项配置]  
  
// [开始消息定义]  
message Msg {  
  uint32 id = 1; //消息 ID  string content = 2;//消息内容  
}  
// [结束消息定义]
```
在文件的头部，需要声明一下目前使用的 proto 协议的版本，proto3 和 proto2 的消息格式有细微的不同，默认的写一个视为 proto2。
Protobuf 支持非常多的语言，其为不同的怨言提供了一些可选的配置选项，使用 option 来进行配置。
	`option java_package = "com.crazymakercircle.netty.protocol";`：在生成 proto 文件中消息的 POJO 和 Builder 的时候，将生成的 Java 代码放入这个选项制定的类路径中
	`option java_outer_classname = "MsgProtos";`：声明的 Java 类的名称
使用 message 来定义消息的结构体，在生成 proto 对应的 Java 代码的时候，每个具体的消息结构体对应一个最终的 Java POJO 类。
	结构体的字段会对应到 Java 类的属性。
	结构体也可以嵌套结构体，其效果类似于 Java 的内部类。
## 通过控制台生成 POJO 和 Builder
完成 .proto 文件定义后，下一步就是生成 POJO 和 Builder 类，有两种方式来生成 Java 类：控制台方式和 Maven 插件的方式。
需要从 https://github.com/protocolbuffers/protobuf/releases 下载对应的转化应用，然后执行下面的命令（执行的根目录是 src）:
```
 protoc --java_out=./main/java/ ./main/java/com/ryan/protobuf/Msg.proto 
```
树形结构：
```
.
├── main
│   ├── java
│   │   └── com
│   │       └── ryan
│   │           ├── json
│   │           │   ├── JsonMsg.java
│   │           │   └── JsonUtil.java
│   │           └── protobuf
│   │               ├── Msg.proto
│   │               └── pojo
│   │                   └── MsgProto.java
│   └── resources
└── test
    └── java
```
## 通过 Maven 插件生成 POJO 和 Builder
使用 protobuf-maven-plugin 插件，可以非常方便地生成消息的 POJO 类和 Builder 类的Java代码。
在 Maven 的 pom 文件中增加此 plugin 插件的配置项，具体如下：
```xml
<plugin>
    <groupId>org.xolstice.maven.plugins</groupId>
    <artifactId>protobuf-maven-plugin</artifactId>
    <version>0.5.0</version>
    <extensions>true</extensions>
    <configuration>
        <!--proto 文件路径-->
        <protoSourceRoot>${project.basedir}/protobuf</protoSourceRoot>
        <!--目标路径-->
        <outputDirectory>${project.build.sourceDirectory}</outputDirectory>
        <!--设置是否在生成 Java 文件之前清空 outputDirectory 的文件-->
        <clearOutputDirectory>false</clearOutputDirectory>
        <!--临时目录-->
        <temporaryProtoFileDirectory>
            ${project.build.directory}/protoc-temp
        </temporaryProtoFileDirectory>
        <!--protoc 可执行文件路径-->
        <protocExecutable>
            ${project.basedir}/protobuf/protoc3.6.1.exe
        </protocExecutable>
    </configuration>
    <executions>
        <execution>
        <goals>
            <goal>compile</goal>
            <goal>test-compile</goal>
        </goals>
        </execution>
    </executions>
</plugin>
```
protobuf-maven-plugin 插件的配置项，具体介绍如下：
- protoSourceRoot：“proto”消息结构体所在文件的路径；
- outputDirectory：生成的POJO类和Builder类的目标路径；
- protocExecutable：protobu f的 Java 代码生成工具的可执行文件的路径。
配置好之后，执行插件的 compile 命令，Java 代码就利索生成了。或者在 Maven 的项目编译时，POJO 类和 Builder 类也会自动生成。
# Protobuf 序列化和反序列化演示案例
在 Maven 的 pom.xml 文件中加上 protobuf 的 Java 运行包的依赖，代码如下：
```java
<dependency>  
    <groupId>com.google.protobuf</groupId>  
    <artifactId>protobuf-java</artifactId>  
    <version>4.28.1</version>  
</dependency>
```
Java 运行时的 potobuf 依赖坐标的版本，“.proto” 消息结构体文件中的 syntax 配置项值（protobuf协议的版本号），以及通过“.proto”文件生成POJO和Builder类的“protoc”可执行文件的版本，这三个版本需要配套一致。

>[!info] 使用 Builder 构造 POJO 对象
```java
public class ProtobufDemo {  
    public static void main(String[] args) {  
        MsgProto.Msg.Builder builder = MsgProto.Msg.newBuilder();  
        builder.setId(100);  
        builder.setContent("hello world");  
        MsgProto.Msg msg = builder.build();  
        System.out.println(msg);  
    }  
}
```
Protobuf 为每个 message 消息结构体生成的 Java 类中，包含了一个 POJO 类、一个 Builder类。
构造POJO消息，首先使用 POJO 类的 newBuilder 静态方法获得一个 Builder 构造者，其次 POJO 每一个字段的值，需要通过 Builder 构造者的 setter 方法去设置。字段值设置完成之后，使用构造者的 build() 方法构造出 POJO 消息对象。

>[!info] 序列化与反序列化：转化为字节数组
```java
public static void main(String[] args) throws InvalidProtocolBufferException {  
    MsgProto.Msg msg = getMsg();  
    // 生成的数组作为数据传输出去  
    byte[] byteArray = msg.toByteArray();  
  
    MsgProto.Msg msg1 = MsgProto.Msg.parseFrom(byteArray);  
    System.out.println("id = " + msg1.getId());  
    System.out.println("content = " + msg1.getContent());  
}
```

>[!info] 序列化与反序列化：输入输出流
```java
public static void main(String[] args) throws IOException {  
    MsgProto.Msg msg = getMsg();  
    ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();  
    // 写出  
    msg.writeTo(byteArrayOutputStream);  
    // 写入  
    MsgProto.Msg msg2 = MsgProto.Msg.parseFrom(new ByteArrayInputStream(byteArrayOutputStream.toByteArray()));  
    System.out.println("id = " + msg2.getId());  
    System.out.println("content = " + msg2.getContent());  
}
```

>[!info] 序列化与反序列化：类 Head-Content 方式
```java
public static void main(String[] args) throws IOException {  
    MsgProto.Msg msg = getMsg();  
    ByteArrayOutputStream byteArrayOutputStream1 = new ByteArrayOutputStream();  
    msg.writeDelimitedTo(byteArrayOutputStream1);  
    MsgProto.Msg msg3 = MsgProto.Msg.parseDelimitedFrom(new ByteArrayInputStream(byteArrayOutputStream1.toByteArray()));  
    System.out.println("id = " + msg3.getId());  
    System.out.println("content = " + msg3.getContent());  
}
```
反序列化时，调用 Protobuf 生成的 POJO 类的 parseDelimitedFrom（InputStream）静态方法，从输入流中先读取 varint32 类型的长度值，然后根据长度值读取此消息的二进制字节，再反序列化得到 POJO 新的实例。
这种方式可以用于异步操作的NIO应用场景中，解决了粘包/半包的问题。
# Protobuf 编解码实践案例
## Netty 内置的 Protobuf 基础编解码器
Netty 内置的基础 Protobuf 编码器/解码器为：ProtobufEncoder 编码器和 ProtobufDecoder 解码器。此外，还提供了一组简单的解决半包问题的编码器和解码器。
### ProtobufEncoder 编码器
```java
public class ProtobufEncoder extends MessageToMessageEncoder<MessageLiteOrBuilder> {  
    @Override  
    protected void encode(ChannelHandlerContext ctx, MessageLiteOrBuilder msg, List<Object> out)  
            throws Exception {  
        if (msg instanceof MessageLite) {  
            out.add(wrappedBuffer(((MessageLite) msg).toByteArray()));  
            return;  
        }  
        if (msg instanceof MessageLite.Builder) {  
            out.add(wrappedBuffer(((MessageLite.Builder) msg).build().toByteArray()));  
        }  
    }  
}
```
翻开 Netty 源代码，我们发现 ProtobufEncoder 的实现逻辑非常简单，直接使用了 Protobuf POJO 实例的 toByteArray() 方法将自身编码成二进制字节。
### ProtobufDecoder 解码器
ProtobufDecoder 和 ProtobufEncoder 相互对应，只不过在使用的时候，ProtobufDecoder 解码器需要指定一个 Protobuf POJO 实例，作为解码的参考原型（prototype），解码时会根据原型实例找到对应的 Parser 解析器，将二进制的字节解码为 Protobuf POJO 实例。
```java
new ProtobufDecoder(MsgProtos.Msg.getDefaultInstance())
```
### ProtobufVarint32LengthFieldPrepender 长度编码器
在 Java NIO 通信中，仅仅使用以上这组编码器和解码器，传输过程中会存在粘包/半包的问题。Netty 也提供了配套的 Head-Content 类型的 Protobuf 编码器和解码器，在二进制码流之前加上二进制字节数组的长度。
这个编码器的作用是，在 ProtobufEncoder 生成的字节数组之前，前置一个 varint32 数字，表示序列化的二进制字节数量或者长度。
### ProtobufVarint32FrameDecoder 长度解码器
ProtobufVarint32FrameDecoder 和 ProtobufVarint32LengthFieldPrepender 相互对应，其作用是，根据数据包中长度域（varint32类型）中的长度值，解码一个足额的字节数组，然后将字节数组交给下一站的解码器 ProtobufDecoder。
==什么是 varint32 类型的长度？ Protobuf 为什么不用 int 这种固定类型的长度呢?==
varint32 是一种紧凑的表示数字的方法，它不是一种固定长度（如 32 位）的数字类型。varint32 它用一个或多个字节来表示一个数字，值越小的数字，使用的字节数越少，值越大，使用的字节数越多。varint32 根据值的大小自动进行收缩，这能减少用于保存长度的字节数。
也就是说，varint32 与 int 类型的最大区别是：varint32 用一个或多个字节来表示一个数字，而 int 是固定长度的数字。varint32 不是固定长度，所以为了更好地减少通信过程中的传输量，消息头中的长度尽量采用 varint 格式。
# 实战：Protobuf 传输
## 服务端案例
为了清晰地演示 Protobuf 传输，下面设计了一个简单的客户端/服务器传输程序：服务器接收客户端的数据包，并解码成 Protobuf 的 POJO；客户端将 Protobuf 的 POJO 编码成二进制数据包，再发送到服务器端。
在服务器端，Protobuf 协议的解码过程如下：
	先使用 Netty 内置的 ProtobufVarint32FrameDecoder，根据 varint32 格式的可变长度值，从入站数据包中解码出二进制 Protobuf 字节码。
	然后，可以使用 Netty 内置的 ProtobufDecoder 解码器将字节码解码成 Protobuf POJO 对象。
	最后，自定义一个 ProtobufBussinessDecoder 解码器来处理 Protobuf POJO 对象。
服务端的实践案例程序代码如下：
```java
public class ProtobufNettyServer {  
    public static void main(String[] args) {  
        NioEventLoopGroup boss = new NioEventLoopGroup();  
        NioEventLoopGroup worker = new NioEventLoopGroup();  
        ChannelInitializer<SocketChannel> socketChannelChannelInitializer = new ChannelInitializer<SocketChannel>() {  
            @Override  
            protected void initChannel(SocketChannel ch) throws Exception {  
                ch.pipeline().addLast(new ProtobufVarint32FrameDecoder());  
                ch.pipeline().addLast(new ProtobufDecoder(MsgProto.Msg.getDefaultInstance()));  
                ch.pipeline().addLast(new ProtobufServerHandler());  
            }  
        };  
        try {  
            ServerBootstrap serverBootstrap = new ServerBootstrap();  
            serverBootstrap.channel(NioServerSocketChannel.class);  
            serverBootstrap.group(boss, worker);  
            serverBootstrap.childHandler(socketChannelChannelInitializer);  
            ChannelFuture bind = serverBootstrap.bind(new InetSocketAddress(8080));  
            bind.sync();  
            bind.channel().closeFuture().sync();  
        } catch (InterruptedException e) {  
            System.out.println(e.getMessage());  
        } finally {  
            boss.shutdownGracefully();  
            worker.shutdownGracefully();  
        }  
    }  
  
	static class ProtobufServerHandler extends ChannelInboundHandlerAdapter {  
	    @Override  
	    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {  
	        MsgProto.Msg msg1 = (MsgProto.Msg) msg;  
	        System.out.println("id = " + msg1.getId());  
	        System.out.println("content = " + msg1.getContent());  
	        super.channelRead(ctx, msg);  
	    }  
	}
}
```
注意一下 Handler 的添加顺序
```java
// 第一个接触到数据的 Decoder：根据 varint32 格式的可变长度值，从入站数据包中解码出二进制 Protobuf 字节码
ch.pipeline().addLast(new ProtobufVarint32FrameDecoder());  
// 第二个接触到数据的 Decoder：可以使用 Netty 内置的 ProtobufDecoder 解码器将字节码解码成 Protobuf POJO 对象
ch.pipeline().addLast(new ProtobufDecoder(MsgProto.Msg.getDefaultInstance()));  
// 第三个接触到数据的 Decoder：
ch.pipeline().addLast(new ProtobufServerHandler());  
```
## 客户端案例
在客户端开始出站之前，需要提前构造好 Protobuf 的 POJO 对象。然后可以使用通道的 write/writeAndFlush 方法，启动出站处理的流水线执行工作。
客户端的出站处理流程中，Protobuf 协议的编码过程
	（1）先使用Netty内置的 ProtobufEncoder，将 Protobuf POJO 对象编码成二进制的字节数组；
	（2）然后，使用Netty内置的 ProtobufVarint32LengthFieldPrepender 编码器，加上 varint32格式的可变长度。
Netty会将完成了编码后的 Length+Content 格式的二进制字节码发送到服务器端。
```java
public class ProtobufNettyClient {  
  
    public static void main(String[] args) {  
        NioEventLoopGroup worker = new NioEventLoopGroup();  
        ChannelInitializer<SocketChannel> socketChannelChannelInitializer = new ChannelInitializer<SocketChannel>() {  
            @Override  
            protected void initChannel(SocketChannel ch) {  
                ch.pipeline().addFirst(new ProtobufEncoder());  
                ch.pipeline().addFirst(new ProtobufVarint32LengthFieldPrepender());  
            }  
        };  
        try {  
            Bootstrap clientBootStrap = new Bootstrap();  
            clientBootStrap.channel(NioSocketChannel.class);  
            clientBootStrap.handler(socketChannelChannelInitializer);  
            clientBootStrap.group(worker);  
            ChannelFuture connect = clientBootStrap.connect(new InetSocketAddress("127.0.0.1", 8080));  
            connect.sync();  
            Channel channel = connect.channel();  
            MsgProto.Msg.Builder builder = MsgProto.Msg.newBuilder();  
            builder.setId(1);  
            builder.setContent("你好服务器，这里是客户端");  
            MsgProto.Msg build = builder.build();  
            channel.writeAndFlush(build);  
        } catch (InterruptedException e) {  
            System.out.println(e.getMessage());  
        } finally {  
            worker.shutdownGracefully();  
        }  
    }  
  
}
```
也是需要注意一下编码器的添加顺序：
```java
// 第一个编码器：将 Protobuf POJO 对象编码成二进制的字节数组
ch.pipeline().addFirst(new ProtobufEncoder());  
// 第二个编码器：加上 varint32格式的可变长度
ch.pipeline().addFirst(new ProtobufVarint32LengthFieldPrepender());  
```
# Protobuf 协议语法详解
[[📂 ProtoBuf 协议语法详解]]