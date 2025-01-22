Netty 的 Decoder 是一个 InBound 入站处理器，解码器负责处理“入站数据”。
它能将上一站 Inbound 入站处理器传过来的输入（Input）数据，进行数据的解码或者格式转换，然后 **发送** 到下一站Inbound入站处理器。
一个标准的解码器的职责为：将输入类型为 ByteBuf 缓冲区的数据进行解码，输出一个一个的 Java POJO 对象。Netty 内置了这么一个抽象的解码器类 ByteToMessageDecoder。
Netty 中的解码器，都是 Inbound 入站处理器类型，几乎都直接或者间接地实现了入站处理的接口 ChannelInboundHandler。
# 原理：ByteToMessageDecoder 解码器处理流程
ByteToMessageDecoder 是一个非常重要的解码器基类，它是一个抽象类，实现了解码处理的基础逻辑和流程。它继承自 ChannelInboundHandlerAdapter.
ByteToMessageDecoder 解码的流程，具体可以描述为：
	首先，它将上一站传过来的输入到 Bytebuf 中的数据进行解码，解码出一个 `List<Object>` 对象列表；
	然后，迭代 `List<Object>` 列表，逐个将 Java POJO 对象传入下一站 Inbound 入站处理器。
![[ByteToMessageDecoder 解码流程.png|1000]]
ByteToMessageDecoder 是个抽象类，不能以实例化方式创建对象。也就是说，直接通过 ByteToMessageDecoder 类，并不能完成 Bytebuf 字节码到具体 Java 类型的解码，还得依赖于它的具体实现。
ByteToMessageDecoder 的解码方法名为 decode，这是一个抽象方法，ByteToMessageDecoder 没有具体的实现。
作为解码器的父类，ByteToMessageDecoder 仅仅提供了一个整体框架：
	它会调用子类的 decode 方法，完成具体的二进制字节解码
	然后会获取子类解码之后的 Object 结果，放入自己内部的结果列表 `List<Object>` 中
	最终，父类会负责将 `List<Object>` 中的元素，==一个一个地==传递给下一个站。
从这个角度来说，ByteToMessageDecoder 在设计上使用了模板模式（Template Pattern）。
ByteToMessageDecoder 的子类要做的，需要从入站 Bytebuf 解码出来的所有 Object 实例，加入到父类的 `List<Object>` 列表中。
如果要实现一个自己的解码器，首先继承 ByteToMessageDecoder 抽象类。然后，实现其基类的 decode 抽象方法，总体来说，流程大致如下：
	（1）首先继承 ByteToMessageDecoder 抽象类。
	（2）然后实现其基类的 decode 抽象方法，将 ByteBuf 到目标 POJO 解码逻辑写入此方法，负责将 Bytebuf 中的二进制数据，解码成一个一个的 Java POJO 对象。
	（3）解码完成后，需要将解码后的 Java POJO 对象，放入 decode 方法的 `List<Object>` 实参中，此实参是父类所传入的解码结果收集容器。
# 实战：自定义 Byte2IntegerDecoder
下面是一个小小的 ByteToMessageDecoder 子类实践案例：整数解码器。其功能是：将 ByteBuf 缓冲区中的字节，解码成 Integer 整数类型。
```java
@Slf4j  
public class Byte2IntegerDecoder extends ByteToMessageDecoder {  
    @Override  
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {  
        while (in.readableBytes() >= 4) {  
            int i = in.readInt();  
            log.info("解码器：{}", i);  
            out.add(i);  
        }  
    }  
}
```
# ReplayingDecoder 解码器
使用上面的 Byte2IntegerDecoder 整数解码器会面临一个问题：需要对 ByteBuf 的长度进行检查，如果有足够的字节，才进行整数的读取。
那么这种长度的判断，是否可以由 Netty 帮忙来完成呢？答案是可以的，可以使用 Netty 的 ReplayingDecoder 类可以省去长度的判断。
ReplayingDecoder 类是 ByteToMessageDecoder 的子类。其作用是：
	在读取 ByteBuf 缓冲区的数据之前，需要检查缓冲区是否有足够的字节。
	若 ByteBuf 中有足够的字节，则会正常读取；反之，如果没有足够的字节，则会停止解码。
改写上一个的整数解码器，使用 ReplayingDecoder 基类编写整数解码器，则可以不用进行长度检测。
```java
@Slf4j  
public class Byte2IntegerReplayingDecoder extends ReplayingDecoder {  
    @Override  
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {  
        int i = in.readInt();  
        log.info("解码出一个整数：{}", i);  
        out.add(i);  
    }  
}
```
ReplayingDecoder 进行长度判断的原理其实很简单：它的内部定义了一个新的二进制缓冲区类，对 ByteBuf 缓冲区进行了装饰，这个类名为 ReplayingDecoderBuffer。
该装饰器的特点是：在缓冲区真正读数据之前，首先进行长度的判断：如果长度合格，则读取数据；否则，抛出 ReplayError。ReplayingDecoder 捕获到 ReplayError 后，会留着数据，等待下一次 IO 事件到来时再读取。
# 整数的分包解码器的实践案例
先介绍一个简单点的例子：整数序列解码，并且将它们==两两一组进行相加==，重点是，解码过程中需要保持发送时的次序。
实现思路是比较简单的，当记录到第一个数字的时候，就把它保存下来，等第二个数字到来的时候执行一次相加的操作，那当前是第几次该如何记录呢？
这就涉及到 ReplayingDecoder 的一个非常重要的属性 —— state 成员属性，该成员属性的作用是保存当前解码器在解码过程中的所处阶段。
在 Netty 源代码中，该属性的定义具体如下：
```java
private final ReplayingDecoderByteBuf replayable = new ReplayingDecoderByteBuf();  
private S state;  
  
/**  
 * Creates a new instance with no initial state (i.e: {@code null}). 
 */
 protected ReplayingDecoder() {  
    this(null);  
}  
  
/**  
 * Creates a new instance with the specified initial state. 
 */
 protected ReplayingDecoder(S initialState) {  
    state = initialState;  
}
```
在上一小节定义的整数解码实例中，使用的是默认的无参数构造器，该构造器初始化 state 成员的值为 null，就是没有用到 state 属性。但是，这一小节，就需要用到 state 成员属性了。
```java
@Slf4j  
public class Byte2IntegerReplayingDecoder  
        extends ReplayingDecoder<Byte2IntegerReplayingDecoder.PHASE> {  
    enum PHASE {  
        PHASE_1, PHASE_2;  
    }  
    Byte2IntegerReplayingDecoder() {  
        super(PHASE.PHASE_1);  
    }  
      
    private int first;  
      
    private int second;  
      
    @Override  
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {  
        switch (state()) {  
            case PHASE_1:  
                first = in.readInt();  
                checkpoint(PHASE.PHASE_2);  
            case PHASE_2:  
                second = in.readInt();  
                Integer sum = first + second;  
                out.add(sum);  
                checkpoint(PHASE.PHASE_1);  
                break;  
            default:   
                break;  
        }  
        int i = in.readInt();  
        log.info("解码出一个整数：{}", i);  
        out.add(i);  
    }  
}
```
checkpoint(...) 方法有两个作用：
	（1）设置state属性的值，更新一下当前的状态。
	（2）还有一个非常大的作用，就是设置“读指针检查点”。
什么是 ReplayingDecoder 的 “读指针” 呢？就是 ReplayingDecoder 提取二进制数据的 ByteBuf 缓冲区的 readerIndex 读指针。
“读指针检查点” 是 ReplayingDecoder 类的另一个重要的成员，它用于暂存内部 ReplayingDecoderBuffer 装饰器缓冲区的 readerIndex 读指针，有点儿类似于 mark 标记。
当读数据时，一旦缓冲区可读的二进制数据不够，缓冲区装饰器 ReplayingDecoderBuffer 在抛出 ReplayError 异常之前，会把 readerIndex 读指针的值，还原到之前通过 checkpoint(...) 方法设置的“读指针检查点”。于是乎，在 ReplayingDecoder 下一次重新读取时，还会从“读指针检查点”的位置开始读取。
```java
catch (Signal replay) {  
    replay.expect(REPLAY);  
    if (ctx.isRemoved()) {  
        break;  
    }  
    int checkpoint = this.checkpoint;  
    if (checkpoint >= 0) {  
        in.readerIndex(checkpoint);  
    } else {    
		break; 
	} 
}
```
# 字符串分包解码器实践案例
同样是分为两个阶段
	第一个阶段是获取一个整数，其代表这个消息的字节长度；
	第二阶段是获取字节数组并且解码出字符串的内容
```java
public class Byte2StringReplayingDecoder extends ReplayingDecoder<Byte2StringReplayingDecoder.PHASE> {  
    enum PHASE {  
        PHASE_1,  
        PHASE_2;  
    }  
  
    private int length;  
  
    private byte[] inBytes;  
  
    public Byte2StringReplayingDecoder() {  
        super(PHASE.PHASE_1);  
    }  
  
    @Override  
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {  
        switch (state()) {  
            case PHASE_1:  
                length = in.readInt();  
                inBytes = new byte[length];  
                checkpoint(PHASE.PHASE_2);  
                break;  
            case PHASE_2:  
                in.readBytes(inBytes, 0, length);  
                String s = new String(inBytes, StandardCharsets.UTF_8);  
                out.add(s);  
                checkpoint(PHASE.PHASE_1);  
                break;  
            default:  
                break;  
        }  
    }  
}
```
配置一个 Handler 在解码之后输出结果：
```java
public class DecoderTestHandler extends ChannelInboundHandlerAdapter {  
    @Override  
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {  
        System.out.println(msg);  
        super.channelRead(ctx, msg);  
    }  
}
```
启动一个客户端，其每次发送消息的时候需要将消息的长度也添加进去：
```java
public class NettyClient {  
    public static void main(String[] args) throws InterruptedException {  
        Bootstrap bootstrap = new Bootstrap();  
        bootstrap.channel(NioSocketChannel.class);  
        bootstrap.group(new NioEventLoopGroup());  
        bootstrap.handler(new ChannelInitializer<Channel>() {  
            @Override  
            protected void initChannel(Channel ch) throws Exception {  
                System.out.println("init");  
            }  
        });  
        ChannelFuture connect = bootstrap.connect(new InetSocketAddress("127.0.0.1", 8080));  
  
  
        String content = "你好服务器，这里是客户端";  
        byte[] bytes = content.getBytes(StandardCharsets.UTF_8);  
        int len = bytes.length;  
  
        connect.sync();  
        Channel channel = connect.channel();  
        for (int i = 0; i < 1000; i++) {  
            ByteBuf buffer = channel.alloc().buffer();  
            // 先写入消息的长度
            buffer.writeInt(len);  
            buffer.writeBytes(bytes);  
            channel.writeAndFlush(buffer);  
        }  
    }  
}
```
输出结果：
![[字符串分包解码器实践案例结果.png|800]]
通过 ReplayingDecoder 解码器，可以正确地解码分包后的 ByteBuf 数据包。但是，在实际的开发中，不太建议继承这个类，原因是：
	（1）不是所有的ByteBuf操作都被 ReplayingDecoderBuffer 装饰类所支持，可能有些 ByteBuf 方法在 ReplayingDecoder 的 decode 实现方法中被使用时就会抛出 ReplayError 异常。
	（2）在数据解码逻辑复杂的应用场景，ReplayingDecoder 在解码速度上相对较差。原因是什么呢？在ByteBuf中长度不够时，ReplayingDecoder 会捕获一个 ReplayError 异常，这时会把 ByteBuf 中的读指针还原到之前的读指针检查点（checkpoint），然后结束这次解析操作，等待下一次 IO 读事件。在网络条件比较糟糕时，一个数据包的解析逻辑会被反复执行多次，此时解析过程是一个消耗 CPU 的操作，所以解码速度上相对较差。所以，ReplayingDecoder 更多的是应用于数据解析逻辑简单的场景。
在数据解析复杂的应用场景，建议使用在前文介绍的解码器 ByteToMessageDecoder 或者其子类（后文介绍），它们会更加合适。
这里继承 ByteToMessageDecoder 基类，实现一个定制的 Header-Content 协议字符串内容解码器，代码如下：
```java
public class StringIntegerHeaderDecoder extends ByteToMessageDecoder {  
    @Override  
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {  
        //可读字节小于 4，消息头还没读满，返回  
        if (in.readableBytes() < 4) {  
            return;  
        }  
        //消息头已经完整  
        //在真正开始从缓冲区读取数据之前，调用 markReaderIndex()设置 mark 标记  
        in.markReaderIndex();  
        int length = in.readInt();  
        //从缓冲区中读出消息头的大小，这会导致 readIndex 读指针变化  
        //如果剩余长度不够消息体，还需要 reset 读指针，下一次从相同的位置处理  
        if (in.readableBytes() < length) {  
            //读指针 reset 到消息头的 readIndex 位置处  
            in.resetReaderIndex();  
            return;  
        }  
        // 读取数据，编码成字符串  
        byte[] inBytes = new byte[length];  
        in.readBytes(inBytes, 0, length);  
        out.add(new String(inBytes, StandardCharsets.UTF_8));  
    }  
}
```
# MessageToMessageDecoder
前面的解码器都是将 ByteBuf 缓冲区中的二进制数据解码成 Java 的普通 POJO 对象。是否存在一些解码器，将一种POJO对象解码成另外一种POJO对象呢？答案是：存在的。
只不过与前面不同的是，在这种应用场景下的 Decoder 解码器，需要继承一个新的Netty解码器基类：`MessageToMessageDecoder<I>`。在继承它的时候，需要明确的泛型实参 `<I>` ，其作用就是指定入站消息的 Java POJO 类型。
`MessageToMessageDecoder<I>` 的入站消息的类型是不明确的，可以是任何的 POJO 类型，所以需要指定。MessageToMessageDecoder 同样使用了模板模式，也有一个 decode 抽象方法，其具体解码的逻辑需要子类去实现。
下面通过实现一个整数 Integer 到字符串 String 转换的解码器，演示一下 MessageToMessageDecoder 的使用。
```java
public class Integer2StringDecoder extends MessageToMessageDecoder<Integer> {
	@Override
	public void decode(ChannelHandlerContext ctx, Integer msg,
		List<Object> out)...{
		out.add(String.valueOf(msg));
	}
}
```
基类泛型实参为 Integer，表明子类解码器的入站的数据类型为 Integer。在 decode 方法中，将整数转成字符串，再加入到一个 List 输出容器中即可，List 容器是由父类在调用时传递过来的。
