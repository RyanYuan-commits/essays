在 Netty 的业务处理完成后，业务处理的结果往往是某个 Java POJO 对象，需要编码成最终的 ByteBuf 二进制类型，通过流水线写入到底层的 Java 通道，这就需要用到 Encoder（编码器）。
编码器是一个 Outbound 出站处理器，负责处理“出站”数据；其次，编码器将上一站 Outbound 出站处理器传过来的输入（Input）数据进行编码或者格式转换，然后传递到下一站 ChannelOutboundHandler 出站处理器。编码器与解码器相呼应，Netty 中的编码器负责将“出站”的某种 Java POJO 对象编码成二进制 ByteBuf，或者转换成另一种 Java POJO 对象。
# MessageToByteEncoder
MessageToByteEncoder 是一个非常重要的编码器基类，它位于 Netty 的 io.netty.handler.codec 包中。
MessageToByteEncoder 的功能是将一个 Java POJO 对象编码成一个 ByteBuf 数据包。它是一个抽象类，仅仅实现了编码的基础流程，在编码过程中，通过调用 encode 抽象方法来完成。
实现 encode 抽象方法的工作需要子类去完成。如果要实现一个自己的编码器，则需要继承自 MessageToByteEncoder 基类，实现它的encode 抽象方法。
```java
public class Integer2ByteEncoder extends MessageToByteEncoder<Integer> {  
    @Override  
    protected void encode(ChannelHandlerContext ctx, Integer msg, ByteBuf out) throws Exception {  
        out.writeInt(msg);  
        System.out.println("encode() 被调用");  
    }  
}
```
在继承 MessageToByteEncoder 时，需要带上泛型实参，具体为编码之前的 Java POJO 原类型（输入类型）。

>[!info] MessageToByteEncoder 使用案例
```java
public class Integer2ByteEncoderServer {  
  
    public static void main(String[] args) {  
        ChannelInitializer<EmbeddedChannel> channelInitializer = new ChannelInitializer<EmbeddedChannel>() {  
            @Override  
            protected void initChannel(EmbeddedChannel ch) throws Exception {  
                ch.pipeline().addFirst(new Integer2ByteEncoder());  
            }  
        };  
        EmbeddedChannel embeddedChannel = new EmbeddedChannel(channelInitializer);  
        for (int i = 0; i < 100; i++) {  
            embeddedChannel.writeOutbound(i);  
        }  
        embeddedChannel.flush();  
        ByteBuf buf = embeddedChannel.readOutbound();  
        while (buf != null) {  
            System.out.println(buf.readInt());  
            buf = embeddedChannel.readOutbound();  
        }  
    }  
  
}  
  
class Integer2ByteEncoder extends MessageToByteEncoder<Integer> {  
    @Override  
    protected void encode(ChannelHandlerContext ctx, Integer msg, ByteBuf out) throws Exception {  
        out.writeInt(msg);  
    }  
}
```
# MessageToMessageEncoder
上一节的示例程序是将 POJO 对象编码成 ByteBuf 二进制对象，那么，是否能够通过 Netty 的编码器将某一种 POJO 对象编码成另外一种的POJO对象呢？需要继承另外一个 Netty 的重要编码器——MessageToMessageEncoder编码器，并实现它的 encode 抽象方法。在子类的 encode 方法实现中，完成原 POJO 类型到目标 POJO 类型的转换逻辑。在 encode 实现方法中，编码完成后，将解码后的目标对象加入到 encode 方法中的实参 List 输出容器即可。
下面是一个从字符串 String 到整数 Integer 的编码器，来演示一下 MessageToMessageEncoder 的使用。
此编码器的具体功能是将字符串中的所有数字提取出来，然后输出到下一站。代码很简单，具体如下：
```java
public class String2IntegerEncoderServer {  
  
    public static void main(String[] args) {  
        ChannelInitializer<EmbeddedChannel> channelInitializer = new ChannelInitializer<EmbeddedChannel>() {  
            @Override  
            protected void initChannel(EmbeddedChannel ch) throws Exception {  
                ch.pipeline().addFirst(new String2IntegerEncoder());  
                ch.pipeline().addFirst(new Integer2ByteEncoder());  
            }  
        };  
        EmbeddedChannel channel = new EmbeddedChannel(channelInitializer);  
        for (int j = 0; j < 100; j++) {  
            String s = "i am " + j;  
            channel.write(s); //向着通道写入含有数字的字符串  
        }  
        channel.flush();  
        ByteBuf buf = channel.readOutbound();  
        while (null != buf) {  
            System.out.println("o = " + buf.readInt()); //打印数字  
            buf = channel.readOutbound(); // 读取数字  
        }  
    }  
  
    static class String2IntegerEncoder extends MessageToMessageEncoder<String> {  
        @Override  
        protected void encode(ChannelHandlerContext ctx, String msg, List<Object> out) throws Exception {  
            char[] charArray = msg.toCharArray();  
            for (char c : charArray) {  
                if (c >= 48 && c <= 57) {  
                    out.add((int) c);  
                }  
            }  
        }  
    }  
  
    static class Integer2ByteEncoder extends MessageToByteEncoder<Integer> {  
        @Override  
        protected void encode(ChannelHandlerContext ctx, Integer msg, ByteBuf out) throws Exception {  
            out.writeInt(msg);  
        }  
    }  
  
  
}w
```