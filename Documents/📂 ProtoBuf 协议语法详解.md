在 Protobuf 中，通信协议的格式是通过 “.proto” 文件定义的。
一个 “.proto” 文件有两大组成部分：头部声明、消息结构体的定义。
	头部声明的部分，主要包含了协议的版本、包名、特定语言的选项设置等；
	消息结构体部分，可以定义一个或者多个消息结构体。
在Java中，当用 Protobuf 编译器来编译 “.proto” 文件时，编译器将生成 Java 语言的 POJO 消息类和 Builder 构造者类。通过 POJO 消息类和 Builder 构造者，Java 程序可以很容易地操作在 .proto 文件中定义的消息和字段：包括获取、设置字段值，将消息序列化到一个输出流中（序列化），以及从一个输入流中解析消息（反序列化）。
# proto 文件的头部声明
简单的 Netty 案例：[[🏀【Netty】使用 Protobuf 通信#一个简单的 proto 文件案例]]，其中对于 proto 文件的头部声明如下：
```java
// [开始头部声明]  
syntax = "proto3";  
package com.ryan.protobuf;  
// [结束头部声明]  
  
// [开始 java 选项配置]  
option java_package = "com.ryan.protobuf.pojo";  
option java_outer_classname = "MsgProto";  
// [结束 java 选项配置]  
```
## syntax 版本号
对于一个 “.proto” 文件而言，文件第一个非空、非注释的行必须注明 Protobuf 的语法版本，这里为 syntax = "proto3"，如果没有声明，则默认版本是 "proto2"。
## package 包
和 Java 语言类似，通过 package 指定包名，用来避免消息（message）名字冲突。如果两个消息的名称相同，但是他们的 package 包名不同，则它们可以共同存在的。
通过 package，还可以实现消息的引用。例如，假设第一个 “.proto” 文件定义了一个 Msg 结构体，package 包名如下：
```protobuf
package com.crazymakercircle.netty.protocol;
message Msg{ ... }
```
假设另一个 “.proto” 文件，也定义了一个相同名字的消息，package 包名如下：
```protobuf
package com.other.netty.protocol;

message Msg{
	// ...
	com.crazymakercircle.netty.protocol.Msg crazyMsg = 1;
	//…
}
```
我们可以看到，在第二个 “.proto” 文件中，可以用 “包名+消息名称”（全限定名）来引用第一个 “.proto” 文件中的Msg结构体，而且不同包中的结构体可以同名。
这一点和 Java 中 package 的使用方法是一样的。另外，package 指定包名后，会对应到生成的消息 POJO 代码和 Builder 代码。在 Java 语言中，会以 package 指定的包名作为生成的POJO类的包名。
## option 选项配置
不是所有的 option 配置选项都会生效，option 选项是否生效与 “.proto” 文件使用的一些特定的语言场景有关。
在Java语言中，以 “java_” 打头的 “option” 选项会生效。
选项 “option java_package” 表示Protobuf编译器在生成Java POJO消息类时，生成在此选项所配置的 Java 包名下。如果没有该选项，则会以头部声明中的 package 作为 Java 包名。
选项 “option java_multiple_files” 表示在生成 Java 类时的打包方式，具体来说，有以下两种方式：
	方式1：一个消息对应一个独立的 Java 类。
	方式2：所有的消息都作为内部类，打包到一个外部类中。
此选项的值，默认为false，也即是方式2，表示使用外部类打包的方式。如果设置“option java_multiple_files= true”，则使用第一种方式生成 Java 类，则一个消息对应一个 POJO Java 类，多个消息结构体会对应到多个类。
选项 “option java_outer_classname” 表示在 Protobuf 编译器在生成 Java POJO 消息类时，如果采用的是上面的方式2（全部POJO类都作为内部类打包在同一个外部类中），则以此选项所配置的值，作为唯一外部类的类名。
# Protobuf 的消息结构体与消息字段
```protobuf
// [开始消息定义]  
message Msg {  
  uint32 id = 1; //消息 ID  string content = 2;//消息内容  
}  
// [结束消息定义]
```
Protobuf 的消息阻断的格式是这样的：
限定修饰符 | 数据类型 | 字段名称 | = ｜ 分配标识号
## 限定修饰符
repeated 限定修饰符：表示该字段可以包含 0~N 个元素值，相当于 Java 中的 List（列表数据类型）。
singular 限定修饰符：表示该字段可以包含 0~1 个元素值。singular 限定修饰符是默认的字段修饰符。
reserved 限定修饰符：指定保留字段名称（Field Name）和分配标识号（Assigning Tags），用于将来的扩展。
下面是一个简单的reserved限定修饰符使用的例子：
```protobuf
message Example {
  reserved 2;
  int32 field = 2; // 错误！编号 2 已被保留。
}
```
## 字段名称
字段名称的命名与 Java 语言的成员变量命名方式几乎是相同的。
Protobuf 建议字段的命名以下划线分割，例如使用 first_name 形式。
## 分配标识号
在消息定义中，每个字段都有唯一的一个数字标识符，可以理解为字段编码值，叫做分配标识号（Assigning Tags）。通过该值，通信双方才能互相识别对方的字段。
当然，相同的编码值，它的限定修饰符和数据类型必须相同。分配标识号是用来在消息的二进制格式中识别各个字段的，一旦开始使用就不能够再改变。
分配标识号的取值范围为 1~232（4294967296）。其中编号`[1，15]`之内的分配标识号，时间和空间效率都是最高的。为什么呢？`[1，15]` 之内的标识号，在编码的时候只会占用一个字节，`[16，2047]`之内的标识号则要占用2个字节。所以那些频繁出现的消息字段，应该使用 `[1，15]` 之内的标识号。
切记：要为将来有可能添加的、频繁出现的字段预留一些标识号。另外，`[1900，2000]` 之内的标识号，为Protobuf内部保留值，建议不要在自己的项目中使用。
标识号的特点是：一个消息结构体中的标识号是可以不连续的；在同一个消息结构体中，不同的字段不能使用相同的标识号。
## 数据类型
| .proto Type | 说明                                          | 对应的 Java Type |
| ----------- | ------------------------------------------- | ------------- |
| double      | 双精度浮点型                                      | double        |
| Float       | 单精度浮点型                                      | float         |
| int32       | 使用变长编码，对于负值的效率很低，如果字段有可能有负值，请使用 sint64 替代   | int           |
| uint32      | 使用变长编码的 32 位整型。                             | int           |
| uint64      | 使用变长编码的 64 位整型。                             | long          |
| sint32      | 使用变长编码，有符号的 32 位整型值。这些编码在负值时比 int32 高效得多。   | int           |
| sint64      | 使用变长编码，有符号的 64 位整型值。编码时比通常的 int64 高效。       | long          |
| fixed32     | 总是 4 个字节，如果数值总是比 2^28 大的话，这个类型会比 uint32 高效。 | int           |
| fixed64     | 总是 8 个字节，如果数值总是比 2^56 大的话，这个类型会比 uint64 高效。 | long          |
| sfixed32    | 总是 4 个字节                                    | int           |
| sfixed64    | 总是 8 个字节                                    | long          |
| Bool        | 布尔型                                         | boolean       |
| String      | 一个字符串必须是 UTF - 8编码或者7 - bit ASCII编码的文本。     | String        |
| Bytes       | 可能包含任意顺序的字节数据。                              | ByteString    |
变长编码的类型（如int32）表示打包的字节并不是固定，而是根据数据的大小或者长度来定。例如 int32，如果数值比较小，在 0~127 时，使用一个字节打包。
关于定长编码（如fixed32）和变长编码（如int32）的区别：fixed32 的打包效率比 int32的效率高，但是使用的空间一般比 int32 多。
因此定长编码属于时间效率高，变长编码属于空间效率高，可以根据项目的实际情况选择。一般情况下可以选择 fixed32，但是遇到对传输效率要求比较苛刻的环境，则可以选择 int32。
# proto 文件的其他语法规范
## import声明
在需要多个消息结构体时，“.poto” 文件可以像 Java 语言的类文件一样，按照模块进行分开设计，所以一个项目可能有多个 “.poto” 文件，一个文件在需要依赖其他 “.poto” 文件的时候，可以通过 import 进行导入。
导入的操作和Java的import的操作大致相同。
## 嵌套消息
“.proto” 文件支持嵌套消息，消息中既可以包含另一个消息实例作为其字段，也可以
```protobuf
// 在消息中定义一个新的消息。
message Outer {
	// Level 0
	message MiddleA {
			// Level 1
			message Inner { // Level 2
				int64 ival = 1;
				bool booly = 2;
			}
	}
	message MiddleB {
		// Level 1
		message Inner { // Level 2
			int32 ival = 1;
			bool booly = 2;
		}
	}
}
```

如果想在父消息类型的外部重复使用这些内部的消息类型，可以使用 Parent.Type 的形式来进行引用，例如：
```java
message SomeOtherMessage {
	Outer.MiddleA.Inner ref = 1;
}
```
## enum 枚举
枚举的定义和 Java 相同，但是有一些限制：枚举值必须大于等于 0 的整数。另外，需要使用分号（;）分隔枚举变量，而不是Java语言中的逗号“,”。
```java
enum VoipProtocol {
	H323 = 1;
	SIP = 2;
	MGCP = 3;
	H248 = 4;
}
```