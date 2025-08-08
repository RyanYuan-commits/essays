# 简明介绍
==(1) 固定长度数据包 Decoder==
其类名为 FixedLengthFrameDecoder，其适用于每次接收到的数据包的长度都是固定的，例如 100 个字节。
在这种场景下，只需要把这个解码器加到流水线中，它会把入站 ByteBuf 数据包拆分成一个个长度为 100 的数据包，然后发往下一个入站处理器。

==(2) 行分割数据包解码器==
其类名为 LineBasedFrameDecoder，适用于每个 ByteBuf 数据包，==使用换行符（或者回车换行符）作为数据包的边界分割符==。
在这种场景下，只需要把这个 LineBasedFrameDecoder 解码器加到流水线中，Netty 会使用换行分隔符，把ByteBuf数据包分割成一个一个完整的应用层 ByteBuf 数据包，再发送到下一站。

==(3) 自定义分个数数据包解码器==
DelimiterBasedFrameDecoder 是 LineBasedFrameDecoder 按照行分割的通用版本。不同之处在于，这个解码器更加灵活，可以自定义分隔符，而不是局限于换行符。
如果使用这个解码器，那么所接收到的数据包，末尾必须带上对应的分隔符。

==(4) 自定义长度数据包解码器==
这是一种基于灵活长度的解码器。在 ByteBuf 数据包中，加了一个长度域字段，保存了原始数据包的长度。解码的时候，会按照这个长度进行原始数据包的提取。
此解码器在所有开箱即用解码器中是最为复杂的一种，后面会重点介绍。
# LineBasedFrameDecoder
在前面字符串分包解码器中，内容是按照 Header-Content 协议进行传输的。
如果不使用 Header-Content 协议，而是在发送端通过换行符（`\n`或者`\r\n`）来分割每一次发送的字符串。 在Netty 中，提供了一个开箱即用的、使用换行符分割字符串的解码器，它的名字为 LineBasedFrameDecoder，它也是一个最为基础的 Netty 内置解码器。
这个解码器的 **工作原理** 很简单，它依次遍历ByteBuf数据包中的可读字节，判断在二进制字节流中，是否存在换行符 `\n` 或者 `\r\n` 的字节码。
	如果有，就以此位置为结束位置，把从可读索引到结束位置之间的字节作为解码成功后的 ByteBuf 数据包。
LineBasedFrameDecoder 支持配置一个最大长度值，表示解码出来的 ByteBuf 最大能包含的字节数。如果连续读取到最大长度后，仍然没有发现换行符，就会抛出异常。
>[!info] 演示 LineBasedFrameDecoder
```java
public class LineBasedFrameDecoderDemo {  
  
    static String split = "\r\n";  
    static String data = "你好服务器，这里是客户端";  
  
    public static void main(String[] args) {  
        ChannelInitializer<EmbeddedChannel> channelInitializer = new ChannelInitializer<EmbeddedChannel>() {  
            @Override  
            protected void initChannel(EmbeddedChannel ch) throws Exception {  
                ch.pipeline().addLast(new LineBasedFrameDecoder(1024));  
                ch.pipeline().addLast(new StringDecoder(StandardCharsets.UTF_8));  
                ch.pipeline().addLast(new OpenBoxHandler());  
            }  
        };  
        EmbeddedChannel embeddedChannel = new EmbeddedChannel(channelInitializer);  
        for (int i = 0; i < 100; i++) {  
            ByteBuf buffer = embeddedChannel.alloc().buffer();  
            // 每次随机发送 1 - 3 个数字  
            int random = new Random().nextInt(3) + 1;  
            for (int j = 0; j < random; j++) {  
                buffer.writeBytes(data.getBytes(StandardCharsets.UTF_8));  
            }  
            buffer.writeBytes(split.getBytes(StandardCharsets.UTF_8));  
            embeddedChannel.writeInbound(buffer);  
        }  
    }  
  
}
```
输出结果：
![[LineBasedFrameDecoder 使用案例.png|800]]
# DelimiterBasedFrameDecoder
DelimiterBasedFrameDecoder 解码器不仅可以使用换行符，还可以将其他的特殊字符作为数据包的分隔符，例如制表符 “\t”。其构造方法如下：
```java
public DelimiterBasedFrameDecoder(
	int maxFrameLength, // 解码的数据包的最大长度
	Boolean stripDelimiter, // 解码后的数据包是否去掉分隔符，一般选择是
	ByteBuf delimiter) // 分隔符
{
	//… 省略构造器的源代码
}
```
DelimiterBasedFrameDecoder 和上面提到的 LineBasedFrameDecoder 使用方式基本相同，只是构造函数不同：
```java
public class DelimiterBasedFrameDecoderDemo {  
  
    static String split = "\t";  
    static String data = "你好服务器，这里是客户端";  
  
    public static void main(String[] args) {  
        ByteBuf delimiter = Unpooled.copiedBuffer(split.getBytes(StandardCharsets.UTF_8));  
        ChannelInitializer<EmbeddedChannel> channelInitializer = new ChannelInitializer<EmbeddedChannel>() {  
            @Override  
            protected void initChannel(EmbeddedChannel ch) throws Exception {  
                // 使用换行符作为分隔符，不使用换行符作为分隔符，则需要使用 DelimiterBasedFrameDecoder                
                ch.pipeline().addLast(new DelimiterBasedFrameDecoder(1024, true, delimiter));  
                ch.pipeline().addLast(new StringDecoder(StandardCharsets.UTF_8));  
                ch.pipeline().addLast(new OpenBoxHandler());  
            }  
        };  
        EmbeddedChannel embeddedChannel = new EmbeddedChannel(channelInitializer);  
        for (int i = 0; i < 100; i++) {  
            ByteBuf buffer = embeddedChannel.alloc().buffer();  
            // 每次随机发送 1 - 3 个数字  
            int random = new Random().nextInt(3) + 1;  
            for (int j = 0; j < random; j++) {  
                buffer.writeBytes(data.getBytes(StandardCharsets.UTF_8));  
            }  
            buffer.writeBytes(split.getBytes(StandardCharsets.UTF_8));  
            embeddedChannel.writeInbound(buffer);  
        }  
    }  
  
}
```
以上实例中，通过 DelimiterBasedFrameDecoder 构造了一个以制表符作为分隔符的字符串分包器。向模拟通道发送字符串的代码，由于与前一小节的发送代码基本相同，这里省略。不过要注意的是，发送一个包后，要发送一个制表符作为结束。
# LengthFieldBasedFrameDecoder
在 Netty 的开箱即用解码器中，最为复杂的是解码器为 LengthFieldBasedFrameDecoder 自定义长度数据包。它的难点在于参数比较多，也比较难以理解。同时它又比较常用，因而下面对它进行重点介绍。
LengthFieldBasedFrameDecoder 可以翻译为“长度域数据包解码器”或者“长度字段数据包解码器”。
传输内容中的 Length Field 长度字段的值，是指存放在数据包中要传输内容的字节数。普通的基于 Header-Content 协议的内容传输，尽量用内置的 LengthFieldBasedFrameDecoder来解码。
一个简单的LengthFieldBasedFrameDecoder使用示例如下：
```java
public class LengthFieldBasedFrameDecoderDemo {  
  
    static String split = "\t";  
    static String data = "你好服务器，这里是客户端";  
  
    public static void main(String[] args) {  
        ByteBuf delimiter = Unpooled.copiedBuffer(split.getBytes(StandardCharsets.UTF_8));  
        ChannelInitializer<EmbeddedChannel> channelInitializer = new ChannelInitializer<EmbeddedChannel>() {  
            @Override  
            protected void initChannel(EmbeddedChannel ch) throws Exception {  
                ch.pipeline().addLast(new LengthFieldBasedFrameDecoder(1024, 0, 4, 0, 4));  
                ch.pipeline().addLast(new StringDecoder(StandardCharsets.UTF_8));  
                ch.pipeline().addLast(new OpenBoxHandler());  
            }  
        };  
        EmbeddedChannel embeddedChannel = new EmbeddedChannel(channelInitializer);  
        for (int i = 0; i < 100; i++) {  
            ByteBuf buffer = embeddedChannel.alloc().buffer();  
            String s = "第" + i + "次发送: " + data;  
            byte[] bytes = s.getBytes(StandardCharsets.UTF_8);  
            int len = bytes.length;  
            buffer.writeInt(len);  
            buffer.writeBytes(bytes);  
            embeddedChannel.writeInbound(buffer);  
        }  
    }  
  
}
```
上面的案例中，使用了这个构造函数：
```java
public LengthFieldBasedFrameDecoder(  
        int maxFrameLength,  // 发送数据包的最大长度
        int lengthFieldOffset,  // 长度字符串的偏移量
        int lengthFieldLength,  // 长度字符串的字节数
        int lengthAdjustment,  // 长度字符串偏移量矫正
        int initialBytesToStrip)  // 丢弃的起始字节数
{  
        ......
}
```
在前面的示例程序中，涉及5个参数和值，分别解读如下：
	（1）maxFrameLength：发送的数据包的最大长度。示例程序中该值为1024，表示一个数据包最多可发送1024个字节。
	（2）lengthFieldOffset：长度字段偏移量。指的是长度字段位于整个数据包内部字节数组中的下标索引值。
	（3）lengthFieldLength：长度字段所占的字节数。如果长度字段是一个int整数，则为4；如果长度字段是一个short整数，则为2。
	（4）lengthAdjustment：长度的矫正值。在传输协议比较复杂的情况下，例如协议包含了长度字段、协议版本号、魔数等等。那么，解码时，就需要进行长度矫正，而这个版本号、魔数等占的字节长度就是这个长度的矫正值。
	（5）initialBytesToStrip：丢弃的起始字节数。在有效数据字段 Content 前面，如果还有一些其他字段的字节，作为最终的解析结果可以丢弃。例如，上面的示例程序中，前面有 4 个节点的长度字段，它起辅助的作用，最终的结果中不需要这个长度，所以丢弃的字节数为 4。
前面的示例程序中，自定义长度解码器的构造参数值如下：
```java
LengthFieldBasedFrameDecoder spliter = new LengthFieldBasedFrameDecoder(1024,0,4,0,4);
```
## 多字段 Head-Content 协议数据解析案例
Head-Content协议是最为简单的内容传输协议。而在实际使用过程中，则没有那么简单，除了长度和内容，在数据包中还可能包含了其他字段，例如，包含了协议版本号。
![[包含协议版本号的Head-Content协议的示意图.png]]
那么，使用 LengthFieldBasedFrameDecoder 解码器，解析以上带有版本号 Head-Content协议报文，参数就需要进行如下的设置：
	maxFrameLength 可以为1024，表示数据包的最大长度为1024个字节。
	lengthFieldOffset 为 0，表示长度字段处于数据包的起始位置。
	lengthFieldLength 实例中的值为4，表示长度字段的长度为4个字节。
	lengthAdjustment 为2，在这个例子中，lengthAdjustment 就是夹在内容字段和长度字段中的部分——版本号的长度。
	initialBytesToStrip 为 6，表示获取最终 Content 内容的字节数组时，抛弃最前面的 6 个字节数据。换句话说，长度字段、版本字段的值被抛弃。
最终写出的代码就是这样的：
```java
public class LengthFieldBasedFrameDecoderDemo {  
  
    static String split = "\t";  
    static String data = "你好服务器，这里是客户端";  
    static byte[] version = "1.0".getBytes();  
  
    public static void main(String[] args) {  
        ByteBuf delimiter = Unpooled.copiedBuffer(split.getBytes(StandardCharsets.UTF_8));  
        ChannelInitializer<EmbeddedChannel> channelInitializer = new ChannelInitializer<EmbeddedChannel>() {  
            @Override  
            protected void initChannel(EmbeddedChannel ch) throws Exception {  
                ch.pipeline().addLast(new LengthFieldBasedFrameDecoder(1024, 0, 4, version.length, 4 + version.length));  
                ch.pipeline().addLast(new StringDecoder(StandardCharsets.UTF_8));  
                ch.pipeline().addLast(new OpenBoxHandler());  
            }  
        };  
        EmbeddedChannel embeddedChannel = new EmbeddedChannel(channelInitializer);  
        for (int i = 0; i < 100; i++) {  
            ByteBuf buffer = embeddedChannel.alloc().buffer();  
            String s = "第" + i + "次发送: " + data;  
            byte[] bytes = s.getBytes(StandardCharsets.UTF_8);  
            int len = bytes.length;  
            buffer.writeInt(len);  
            buffer.writeBytes(version);  
            buffer.writeBytes(bytes);  
            embeddedChannel.writeInbound(buffer);  
        }  
    }  
  
}
```
特别注意一下这两个位置的修改：
```java
// 新增 version 字段
static byte[] version = "1.0".getBytes();  

// 构造函数的改变
ch.pipeline().addLast(new LengthFieldBasedFrameDecoder(1024, 0, 4, version.length, 4 + version.length));  

// 传输数据的时候写入 version 字段
buffer.writeBytes(version);  
```