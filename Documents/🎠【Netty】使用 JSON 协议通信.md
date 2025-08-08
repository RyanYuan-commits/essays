>JSON(JavaScript Object Notation, JS 对象简谱) 是一种轻量级的数据交换格式。它基于 ECMAScript (欧洲计算机协会制定的JS规范)的一个子集，采用完全独立于编程语言的文本格式来存储和表示数据。
>简洁和清晰的层次结构使得JSON成为理想的数据交换语言。JSON 协议是一种文本协议，易于人阅读和编写，同时也易于机器解析和生成，并能有效地提升网络传输效率。
# JSON 的核心优势
XML 也是一种常用的文本协议，XML 和 JSON 都使用结构化方法来标记数据。
==和 XML 相比，JSON 作为数据包格式传输的时候具有更高的效率==，这是因为 JSON 不像 XML 那样需要有严格的闭合标签，这就让有效数据量与总数据包比大大提升，从而同等数据流量的情况下，JSON 减少网络的传输压力。
做一个简单的比较，用 XML 表示中国部分省市的数据如下：
```xml
<?xml version="1.0" encoding="utf-8"?>
<country>
    <name>中国</name>
    <province>
        <name>广东</name>
        <cities>
            <city>广州</city>
            <city>深圳</city>
        </cities>
    </province>
    <province>
        <name>新疆</name>
        <cities>
            <city>乌鲁木齐</city>
        </cities>
    </province>
</country>
```
而使用 JSON 来表示：
```json
{
    "name": "中国",
    "province": [
        {
            "name": "广东",
            "cities": {
                "city": ["广州", "深圳"]
            }
        }, 
        {
            "name": "新疆",
            "cities": {
                "city": ["乌鲁木齐"]
            }
        }
    ]
}
```
JSON 的语法格式和清晰的层次结构非常简单，明显要比 XML 容易阅读，并且在数据交换方面，由于 JSON 所使用的字符要比 XML 少得多，可以大大得节约传输数据所占用的带宽。
# JSON 的序列化和反序列化开源库
Java 处理 JSON 数据有三个比较流行的开源类库有：阿里的 FastJson、谷歌的 Gson 和开源社区的 Jackson。
Jackson 是一个简单的、基于 Java 的 JSON 开源库。使用 Jackson 开源库，可以轻松地将 Java POJO 对象转换成 JSON、XML 格式字符串；同样也可以方便地将 JSON、XML 字符串转换成 Java POJO 对象。
	Jackson 开源库的优点是：所依赖的 Jar 包较少、简单易用、性能也还不错，另外 Jackson 社区相对比较活跃。
	Jackson 开源库的缺点是：对于复杂POJO类型、复杂的集合 Map、List 的转换结果，不是标准的 JSON 格式，或者会出现一些问题。
	
Google 的 Gson 开源库是一个功能齐全的 JSON 解析库，起源于 Google 公司内部需求而由 Google 自行研发而来，在 2008 年 5 月公开发布第一版之后已被许多公司或用户应用。
Gson 可以完成复杂类型的 POJO 和 JSON 字符串的相互转换，转换的能力非常强。

阿里巴巴的 FastJson 是一个高性能的 JSON 库。
	顾名思义，FastJson 库采用独创的快速算法，将 JSON 转成 POJO 的速度提升到极致，从性能上说，其反序列化速度超过其他 JSON 开源库。
	传闻说 FastJson 在复杂类型的 POJO 转换 JSON（序列化）时，可能会出现一些引用类型问题而导致 JSON 转换出错，需要进行引用的定制。

在实际开发中，目前主流的策略是：Google 的 Gson 库和阿里的 FastJson 库两者结合使用。
	在 POJO 序列化成 JSON 字符串的应用场景（序列化场景），使用 Google 的 Gson 库；
	在 JSON 字符串反序列化成 POJO 的应用场景（反序列化场景），使用阿里的 FastJson 库。
```java
public class JsonUtil {  
  
    static GsonBuilder gb = new GsonBuilder();  
    static {  
        // 不需要 html escape        gb.disableHtmlEscaping();  
    }  
      
    public static String pojoToJson(Object object) {  
        String json = gb.create().toJson(object);  
        return json;  
    }  
      
    public static <T> T jsonToPojo(String json, Class<T> tClass){  
        T t = JSONObject.parseObject(json, tClass);  
        return t;  
    }  
  
}
```
# JSON 传输的编码器和解码器
本质上来说，JSON 格式仅仅是字符串的一种组织形式。所以，传输 JSON 的所用到的协议与传输普通文本所使用的协议没有什么不同。下面使用常用的 Head-Content 协议来介绍一下 JSON 传输。
![[JSON格式Head-Content数据包的解码过程.png|700]]
先使用 Netty 内置的 LengthFieldBasedFrameDecoder 解码 Head-Content 二进制数据包，解码出 Content 字段的二进制内容。然后，使用 StringDecoder 字符串解码器将二进制内容解码成 JSON 字符串。最后，使用自定义业务解码器 JsonMsgDecoder 解码器将 JSON 字符串解码成自定义的 POJO 业务对象。
JSON 的编码过程就是把 JSON String 转化为二进制字节数组，然后使用内置的 LengthFieldPrepender 将其转化为 Head-Content 二进制数据包：
![[JSON格式Head-Content数据包的编码过程.png|800]]
Netty 内置 LengthFieldPrepender 编码器的作用：在数据包的前面加上内容的二进制字节数组的长度。这个编码器和 LengthFieldBasedFrameDecoder 解码器是天生的一对，常常配套使用。
```java
public LengthFieldPrepender(int lengthFieldLength) {
    this(lengthFieldLength, false);
}

public LengthFieldPrepender(int lengthFieldLength, // Head 长度字段占用的字节数目
    Boolean lengthIncludesLengthFieldLength) { // Head 字段的总长度是否包括长度字段自身的字节数
    this(lengthFieldLength, 0, lengthIncludesLengthFieldLength);
}
```
lengthIncludesLengthFieldLength 表示 Head 字段的总长度是否包括长度字段自身的字节数
	为 true 的话，len 最终的结果就是 len 的字节长度 + content 的字节长度
	为 false 的话，len 只表示 content 的字节长度
