---
type: Java 基础
sub-type: 并发编程
finished: "true"
---

## 1 Channel 简介

### 1.1 什么是 Channel

在 OIO 中, 同一个网络连接会关联到输入流和输出流, Java 应用程序通过这两个流进行输入和输出. 

在 NIO 中, 一个网络连接使用一个 `Channel` 表示, 所有的 NIO 的 IO 操作都是通过连接通道完成的. `Channel` 类似于 OIO 中的两个流的结合体, 既可以从通道读取数据, 也可以向通道写入数据. 

![[channel 与 Buffer 读写.png|600]]

### 1.2 Channel 的本质

要清楚的回答这个问题, 还得回到TCP/IP协议的四层模型的基础知识. 具体如下图所示:


![[Java NIO 收发 HTTP 原理图.png|800]]

在 TCP/IP 协议四层模型的最底层为链路层. 在最原始的物理链路时代, 数据传输的两头会通过拉同轴电缆的方式, 拉一条物理电缆, 这条网线就代表一个双向的连接, 通过这条电缆, 双方可以完成数据的传输. 数据传输一旦完成, 需要把这条物理链路拆除. 

>[!question] 而在操作系统的维度, 该怎么标识这种底层的物理链路呢？或者, 操作系统该怎么标识这种底层的虚拟链路呢？
> Linux 系统一切都是文件描述符(File Descriptor). 所以, 这种底层的物理链路, 在操作系统层面, 就会为应用创建一个文件描述符. 
> 
>这点和 Java 里边的对象类似, 一个 Java 对象有内存的数据结构和内存地址, 那么, 一个文件描述符也有一个内核的数据结构和一个进程内的唯一编号来表示. 
>
>然后, 操作系统会把这个文件描述提供给应用层, 应用层通过对这个文件描述符(去对传输链路进行数据的读取和写入. 

NIO 中的 `SocketChannel` 是对底层的传输链路所对应的文件描述符的一种封装:

```java
class SocketChannelImpl extends SocketChannel implements SelChImpl  {  
	/* 文件描述符对象 */
	private final FileDescriptor fd;
}
public final class FileDescriptor { 
	/* 文件描述符 的进程内的唯一编号*/
	private int fd;
}
```

两个 Java 应用通过 NIO 建立双向的连接, 它们各自都会有一个自己内部的文件描述符, 代表这条连接的自己一方：

![[两个 Java 通过 NIO 连接.png|800]]

## 2 Channel 类详解

Java NIO 中, 一个 socket 连接使用一个 Channel 来表示. 然而, 从更广泛的层面来说, 一个通道封装了一个底层的文件描述符, 例如硬件设备、文件、网络连接等. 所以, 与文件描述符相对应, Java NIO 的通道分为很多类型. 

| 通道类型                  | 描述                                                    |
| --------------------- | ----------------------------------------------------- |
| `FileChannel`         | 文件通道, 用于文件的数据读写.                                      |
| `SocketChannel`       | 套接字通道, 用于Socket套接字 TCP 连接的数据读写.                       |
| `ServerSocketChannel` | 服务器套接字通道, 允许我们监听 TCP 连接请求, 为每个连接创建一个 `SocketChannel`. |
| `DatagramChannel`     | 数据报通道, 用于 UDP 协议的数据读写.                                |

这个四种通道, 涵盖了文件 IO、TCP 网络、UDP IO 三类基础 IO 读写操作.

### 2.1 FileChannel 文件通道

`FileChannel` 是专门操作文件的通道. 通过 `FileChannel`, 既可以从一个文件中读取数据, 也可以将数据写入到文件中. 

特别申明一下, `FileChannel` 为阻塞模式, 不能设置为非阻塞模式. 

获取:

```java
static void getChannel() throws FileNotFoundException {  
    File file = new File("file.txt");  
    /* 方案一：通过文件流获取 */ 
    FileInputStream fileInputStream = new FileInputStream(file);  
    // 获取文件流的通道  
    FileChannel inputStreamChannel = fileInputStream.getChannel();  
  
    // 创建一个文件输出流  
    FileOutputStream fileOutputStream = new FileOutputStream(file);  
    // 获取文件流的通道  
    FileChannel outputStreamChannel = fileOutputStream.getChannel();  
  
    /* 方案二：通过 RandomAccessFile, 可读可写 */    
    RandomAccessFile randomAccessFile = new RandomAccessFile(file, "rw");  
    FileChannel randomAccessFileChannel = randomAccessFile.getChannel();  
}
```


读取:

```java
public class FileChannelDemo {  
    public static void main(String[] args) throws IOException, URISyntaxException {  
        File file = new File(FileChannelDemo.class.getResource("/file.txt").toURI());  
        FileInputStream fileInputStream = new FileInputStream(file);  
        FileChannel fileChannel = fileInputStream.getChannel();  
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);  
        while (fileChannel.read(byteBuffer) != -1) {  
            byteBuffer.flip();  
            while (byteBuffer.hasRemaining()) {  
                System.out.print((char) byteBuffer.get() + " ");  
            }  
            byteBuffer.clear();  
        }  
    }  
}
```


写入:

```java
public class FileChannelDemo {  
    public static void main(String[] args) throws IOException, URISyntaxException {  
	    // 获取 File 对象
        File file = new File(FileChannelDemo.class.getResource("/file.txt").toURI());  
        File fileCopy = new File(FileChannelDemo.class.getResource("/file_copy.txt").toURI());  
        
		// 读取文件
        FileInputStream fileInputStream = new FileInputStream(file);  
        FileChannel fileChannel = fileInputStream.getChannel();  
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);  
        int read = 0;  
        while (read != -1) {  
            read = fileChannel.read(byteBuffer);  
        }  

		// 写入文件
        byteBuffer.flip();  
        FileOutputStream fileOutputStream = new FileOutputStream(fileCopy);  
        FileChannel fileChannelCopy = fileOutputStream.getChannel();  
        int write = 1;  
        while (write != 0) {  
            write = fileChannelCopy.write(byteBuffer);  
        }  ;
    }  
}
```


当通道使用完成后, 必须将其关闭, 即调用 close 方法:

```java
channel.close();
```


在将缓冲区写入通道时, 出于性能原因, 操作系统不可能每次都实时将写入数据落地(或刷新)到磁盘, 完成最终的数据保存. 

如果在将缓冲数据写入通道时, 需要保证数据能落地写入到磁盘, 可以在写入后调用一下FileChannel的force()方法. 

```java
//强制刷新到磁盘
channel.force(true);
```


使用 FileChannel 完成文件复制的实践案例:

```java
static void fileCopy(File sourceFile, File destFile) {  
    try {  
        // 如果目标文件不存在, 则新建  
        if ((!destFile.exists())) {  
            boolean newFile = destFile.createNewFile();  
        }  
        FileInputStream fis = null;  
        FileOutputStream fos = null;  
        FileChannel inChannel = null;  
        FileChannel outChanel = null;  
        try {  
            fis = new FileInputStream(sourceFile);  
            fos = new FileOutputStream(destFile);  
            inChannel = fis.getChannel();  
            outChanel = fos.getChannel();  
            // 新建一个 Buffer            inChannel.transferTo(0, inChannel.size(), outChanel);  
        } catch (IOException e) {  
            e.printStackTrace();  
        }  
    } catch (IOException e) {  
        throw new RuntimeException(e);  
    }  
}
```

### 2.2 SocketChannel 套接字通道

在 NIO 中, 涉及网络连接的通道有两个：一个是 `SocketChannel` 负责连接的数据传输, 另一个是 `ServerSocketChannel` 负责连接的监听. 

- `SocketChannel` 与 OIO 中的 `Socket` 类对应. 
- `ServerSocketChannel` 监听通道, 与 OIO 中的 `ServerSocket` 类对应.

`ServerSocketChannel` 仅仅应用于服务器端, 而 `SocketChannel` 则同时处于服务器端和客户端, 所以, 对应于一个连接, 两端都有一个负责传输的 `SocketChannel` 传输通道. 

`ServerSocketChannel` 和 `SocketChannel`, 都支持阻塞和非阻塞两种模式. 

```java
socketChannel.configureBlocking(false); // 设置为非阻塞模式
socketChannel.configureBlocking(true); // 设置为阻塞模式
```

在阻塞模式下, `SocketChannel` 通道的各种操作, 都是同步且阻塞的, 在效率上与 OIO 相同. 在非阻塞模式下, 通道的操作是异步、高效率的, 这也是相对于传统的 OIO 的优势所在.

**I 获取 SocketChannel 通道**

在客户端, 通过 SocketChannel 的静态方法 open 获取一个套接字传输通道, 然后将 socket 套接字设置为非阻塞模式, 最后, 通过 connect 方法, 对服务器的 IP 和端口发起连接. 

```java
// 获得一个套接字通道  
SocketChannel sc = SocketChannel.open();  
// 切换到非阻塞模式  
sc.configureBlocking(false);  
// 对服务器的 IP 和端口发起连接  
sc.connect(new InetSocketAddress("127.0.0.1", 8080));
```

非阻塞的情况下, 与服务器的连接可能还没有真的建立, `sc.connect` 方法就返回了, 因此需要不断的自旋, 检查当前是否连接到了主机. 

在服务器端, 在连接建立的事件到来时, 服务器端的 `ServerSocketChannel` 能成功地查询出这个新连接事件, 并且通过调用服务器端 `ServerSocketChannel` 监听套接字的 `accept` 方法, 来获取新连接的套接字通道：

```java
//新连接事件到来, 首先通过事件, 获取服务器监听通道
ServerSocketChannel server = (ServerSocketChannel) key.channel();

//获取新连接的套接字通道
SocketChannel socketChannel = server.accept();

//设置为非阻塞模式
socketChannel.configureBlocking(false);
```


**II 读取 SocketChannel 传输通道**

当 `SocketChannel` 可读的时候, 可以从 `SocketChannel` 读取数据, 具体方法和恰面问价能听到的读取方法是相同的. 

```java
ByteBufferbuf = ByteBuffer.allocate(1024);
int bytesRead = socketChannel.read(buf);
```

在读取时需要检查 `read` 的返回值, 以便判断当前是否读取到了数据. `read` 方法的返回值是读取的字节数, 如果返回 -1, 那么表示读取到对方的输出结束标志, 对方已经输出结束, 准备关闭连接. 

实际上, 通过 `read` 方法读数据, 本身是很简单的, 比较困难的是, 在非阻塞模式下, 如何知道通道何时是可读的呢？这就需要用到 NIO 的 `Selector` 通道选择器. 


**III 写入到 SocketChannel 传输通道**

```java
// 写入前需要读取缓冲区, 要求 ByteBuffer 是读取模式
buffer.flip();
socketChannel.write(buffer);
```


**IV 关闭 SocketChannel 通道**

在关闭 `SocketChannel` 传输通道前, 如果传输通道用来写入数据, 则建议调用一次 `shutdownOutput` 终止输出方法, 向对方发送一个输出的结束标志(-1). 

然后调用 `socketChannel.close` 方法, 关闭套接字连接. 

```java
// 调用终止输出方法, 向对方发送一个输出的结束标志
socketChannel.shutdownOutput();

// 关闭套接字连接
IOUtil.closeQuietly(socketChannel);
```


**V 通过 SocketChannel 发送文件的实践案例**

```java
public class SocketChannelDemo {  
    public static void main(String[] args) throws IOException {  
        SocketChannel socketChannel = SocketChannel.open();  
        socketChannel.configureBlocking(false);  
        try {  
            File file = new File(FileChannelDemo.class.getResource("/file.txt").toURI());  
            FileChannel channel = new FileInputStream(file).getChannel();  
              
            socketChannel.connect(new InetSocketAddress("localhost", 8080));  
            while (!socketChannel.finishConnect()) {  
                System.out.println("正在连接服务器...");  
            }  
            System.out.println("成功连接到服务器...");  
  
            // 发送文件  
            ByteBuffer buf = ByteBuffer.allocate(1024);  
            // 将数据读取到 buf 中  
            int read = 0;  
            while (read != -1) {  
                read = channel.read(buf);  
            }  
            System.out.println("文件已经成功读取到缓冲区...");  
            buf.flip();  
            socketChannel.write(buf);  
        } catch (URISyntaxException e) {  
            throw new RuntimeException(e);  
        } finally {  
            System.out.println("关闭通道...");  
            socketChannel.close();  
        }  
    }  
}
```

### 2.3 DatagramChannel 数据报通道

在 Java 中使用 UDP 协议传输数据, 比 TCP 协议更加简单. 和 Socket 套接字的 TCP 传输协议不同, UDP协议不是面向连接的协议, 只要知道服务器的IP和端口, 就可以直接向对方发送数据. 

在 Java NIO 中, 使用 `DatagramChannel` 数据报通道来处理UDP协议的数据传输. 

**I 获取 DatagramChannel 数据报通道**

获取数据报通道的方式很简单, 调用 `DatagramChannel` 类的 `open` 静态方法即可. 然后调用 `configureBlocking` 方法, 设置成非阻塞模式. 

```java
//获取 DatagramChannel 数据报通道
DatagramChannel channel = DatagramChannel.open();

//设置为非阻塞模式
datagramChannel.configureBlocking(false);
```

如果需要接受数据, 还需要调用 `bind` 方法绑定一个数据报的监听端口

```java
//调用 bind 方法绑定一个数据报的监听端口
channel.socket().bind(new InetSocketAddress(18080));
```


**II 读取 DatagramChannel 数据报通道的数据**

当 `DatagramChannel` 通道可读时, 可以从 `DatagramChannel` 读取数据. 

和前面的 `SocketChannel` 读取方式不同, 这里不调用 `read` 方法, 而是调用 `receive` 方法将数据从 `DatagramChannel` 读入, 再写入到 `ByteBuffer` 缓冲区中. 

```java
//创建缓冲区
ByteBuffer buf = ByteBuffer.allocate(1024);

//从 DatagramChannel 读入, 再写入到 ByteBuffer 缓冲区
SocketAddress clientAdd r= datagramChannel.receive(buf);
```

`receive` 方法的返回值是 `SocketAddress` 类型, 表示返回发送端的连接地址(包括IP和端口). 


**III 写入 DatagramChannel 数据报通道**

```java
// 把缓冲区翻转到读取模式
buffer.flip();
// 调用 send 方法, 把数据发送到目标 IP+端口
dChannel.send(buffer, new InetSocketAddress("127.0.0.1",18899));
// 清空缓冲区, 切换到写入模式
buffer.clear();
```


**IV 关闭 DatagramChannel 通道**

```java
dChannel.clse();
```