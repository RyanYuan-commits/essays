# Channel 简介
Channel，国内大多翻译成“通道”。Channel 的角色和 OIO 中的Stream(流)是差不多的。
在 OIO 中，同一个网络连接会关联到两个流：一个输入流（Input Stream），另一个输出流（Output Stream），Java应用程序通过这两个流，不断地进行输入和输出的操作。

在NIO中，一个网络连接使用一个Channel（通道）表示，所有的NIO的IO操作都是通过连接通道完成的。
一个通道类似于OIO中的两个流的结合体，既可以从通道读取数据，也可以向通道写入数据。
![[channel 与 Buffer 读写.png|700]]

Channel和Stream的一个显著的不同是：Stream是单向的，譬如InputStream是单向的只读流，OutputStream是单向的只写流；
而Channel是双向的，既可以用来进行读操作，又可以用来进行写操作。
## 什么是 Channel 的本质？
要清楚的回答这个问题，还得回到TCP/IP协议的四层模型的基础知识。具体如下图所示
![[Java NIO 收发 HTTP 原理图.png|900]]
在TCP/IP协议四层模型的最底层为链路层。在最原始的物理链路时代，咱们数据传输的两头（发送方和接收方）会通过拉同轴电缆的方式，拉一条物理电缆（类似于后来更加高级的网线），这条网线就代表一个双向的连接（connection），通过这条电缆，双方可以完成数据的传输。数据传输一旦完成，需要把这条物理链路拆除（就是这么粗暴）。

>[!question] 而在操作系统的维度，该怎么标识这种底层的物理链路呢？或者，操作系统该怎么标识这种底层的虚拟链路呢？
> 前面讲到，操作系统一切都是文件描述符（file descriptor）。所以，这种底层的物理链路，在操作系统层面，就会为应用创建一个文件描述符（file descriptor）。
>这点和 Java 里边的对象类似，一个 Java 对象有内存的数据结构和内存地址，那么，一个文件描述符（file descriptor）也有一个内核的数据结构和一个进程内的唯一      
> 编号来表示。然后，操作系统会把这个文件描述提供给应用层，应用层通过对这个文件描述符（file descriptor）去对传输链路进行数据的读取和写入。

NIO 中的 SocketChannel，实际上就是对底层的传输链路所对应的文件描述符（file descriptor）的一种封装，具体的代码如下：
```java
class SocketChannelImpl  extends SocketChannel  implements SelChImpl  {  
	/* 文件描述符对象 */
	private final FileDescriptor fd;
}
public final class FileDescriptor { 
	/* 文件描述符 的进程内的唯一编号*/
	private int fd;
}
```
如果两个Java应用通过NIO建立双向的连接（传输链路），它们各自都会有一个自己内部的文件描述符（file descriptor），代表这条连接的自己一方：
![[两个 Java 通过 NIO 连接.png|800]]
# Channel 类详解
Java NIO 中，一个 socket 连接使用一个 Channel（通道）来表示。然而，从更广泛的层面来说，一个通道封装了一个底层的文件描述符，例如硬件设备、文件、网络连接等。所以，与文件描述符相对应，Java NIO 的通道分为很多类型。
但是Java的通道更加的细化，例如，对应到不同的网络传输协议类型，在 Java 中都有不同的 NIO Channel（通道）相对应。
## Channel 的主要类型
这里不对Java NIO全部通道类型进行过多的描述，仅仅聚焦于介绍其中最为重要的四种 Channel（通道）实现：FileChannel、SocketChannel、ServerSocketChannel、DatagramChannel。
对于以上四种通道，说明如下：
	（1）FileChannel 文件通道，用于文件的数据读写；
	（2）SocketChannel 套接字通道，用于Socket套接字TCP连接的数据读写；
	（3）ServerSocketChannel 服务器套接字通道（或服务器监听通道），允许我们监听TCP连接请求，为每个监听到的请求，创建一个SocketChannel套接字通道；
	（4）DatagramChannel 数据报通道，用于UDP协议的数据读写。
这个四种通道，涵盖了文件IO、TCP网络、UDP IO三类基础IO读写操作。下面从通道的获取、读取、写入、关闭四个重要的操作入手，对四种通道进行简单的介绍。
## FileChannel 文件通道
FileChannel是专门操作文件的通道。通过FileChannel，既可以从一个文件中读取数据，也可以将数据写入到文件中。
特别申明一下，FileChannel为阻塞模式，不能设置为非阻塞模式。
下面分别对 FileChannel 的获取、读取、写入和关闭四个操作
### FileChannel 的获取
```java
static void getChannel() throws FileNotFoundException {  
    File file = new File("file.txt");  
    /* 方案一：通过文件流获取 */    // 创建一个文件输入流  
    FileInputStream fileInputStream = new FileInputStream(file);  
    // 获取文件流的通道  
    FileChannel inputStreamChannel = fileInputStream.getChannel();  
  
    // 创建一个文件输出流  
    FileOutputStream fileOutputStream = new FileOutputStream(file);  
    // 获取文件流的通道  
    FileChannel outputStreamChannel = fileOutputStream.getChannel();  
  
    /* 方案二：通过 RandomAccessFile */    RandomAccessFile randomAccessFile = new RandomAccessFile(file, "rw");  
    FileChannel randomAccessFileChannel = randomAccessFile.getChannel(); // 可读可写  
}
```
### 读取 FileChannel 通道
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
### 写入 FileChannel 通道
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
        }  
    }  
}
```
### 关闭通道
当通道使用完成后，必须将其关闭，即调用 close 方法
```java
channel.close();
```
### 强制刷新到磁盘
在将缓冲区写入通道时，出于性能原因，操作系统不可能每次都实时将写入数据落地（或刷新）到磁盘，完成最终的数据保存。
如果在将缓冲数据写入通道时，需要保证数据能落地写入到磁盘，可以在写入后调用一下FileChannel的force()方法。
```java
//强制刷新到磁盘
channel.force(true);
```
### 使用 FileChannel 完成文件复制的实践案例
```java
static void fileCopy(File sourceFile, File destFile) {  
    try {  
        // 如果目标文件不存在，则新建  
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
## SocketChannel 套接字通道
在 NIO 中，涉及网络连接的通道有两个：一个是SocketChannel负责连接的数据传输，另一个是ServerSocketChannel负责连接的监听。
其中，NIO 中的 SocketChannel 传输通道，与 OIO 中的 Socket 类对应；NIO 中的 ServerSocketChannel 监听通道，对应于 OIO 中的 ServerSocket 类。
ServerSocketChannel仅仅应用于服务器端，而 SocketChannel 则同时处于服务器端和客户端，所以，**对应于一个连接，两端都有一个负责传输的 SocketChannel 传输通道**。
无论是 ServerSocketChannel，还是 SocketChannel，都支持阻塞和非阻塞两种模式。
```java
socketChannel.configureBlocking(false); // 设置为非血色模式
socketChannel.configureBlocking(true); // 设置为阻塞模式
```
在阻塞模式下，SocketChannel通道的connect连接、read读、write写操作，都是同步的和阻塞式的，在效率上与Java旧的OIO的面向流的阻塞式读写操作相同。
因此，在这里不介绍阻塞模式下的通道的具体操作。在非阻塞模式下，通道的操作是异步、高效率的，这也是相对于传统的OIO的优势所在。下面仅仅详细介绍在非阻塞模式下通道的打开、读写和关闭操作等操作。
### 获取 SocketChannel 通道
在客户端，通过 SocketChannel 的静态方法 open 获取一个套接字传输通道，然后将 socket 套接字设置为非阻塞模式，最后，通过 connect 方法，对服务器的 IP 和端口发起连接。
```java
// 获得一个套接字通道  
SocketChannel sc = SocketChannel.open();  
// 切换到非阻塞模式  
sc.configureBlocking(false);  
// 对服务器的 IP 和端口发起连接  
sc.connect(new InetSocketAddress("127.0.0.1", 8080));
```
非阻塞的情况下，与服务器的连接可能还没有真的简历，sc.connect 方法就返回了，因此需要不断的自旋，检查当前是否连接到了主机。

在服务器端，在连接建立的事件到来时，服务器端的 ServerSocketChannel 能成功地查询出这个新连接事件，并且通过调用服务器端ServerSocketChannel监听套接字的accept()方法，来获取新连接的套接字通道：
```java
//新连接事件到来，首先通过事件，获取服务器监听通道
ServerSocketChannel server = (ServerSocketChannel) key.channel();

//获取新连接的套接字通道
SocketChannel socketChannel = server.accept();

//设置为非阻塞模式
socketChannel.configureBlocking(false);
```
### 读取 SocketChannel 传输通道
当 SocketChannel 可读的时候，可以从 SocketChannel 读取数据，具体方法和恰面问价能听到的读取方法是相同的。
```java
ByteBufferbuf = ByteBuffer.allocate(1024);
int bytesRead = socketChannel.read(buf);
```
在读取时，因为是异步的，因此我们必须检查 read 的返回值，以便判断当前是否读取到了数据。
read() 方法的返回值是读取的字节数，如果返回 -1，那么表示读取到对方的输出结束标志，对方已经输出结束，准备关闭连接。实际上，通过 read 方法读数据，本身是很简单的，比较困难的是，在非阻塞模式下，如何知道通道何时是可读的呢？这就需要用到NIO的新组件——Selector通道选择器，稍后介绍。
### 写入到 SocketChannel 传输通道
和前面吧数据写入到 FileChannel 相同，大部分场景都会调用通道的的 write 方法。
```java
// 写入前需要读取缓冲区，要求 ByteBuffer 是读取模式
buffer.flip();
socketChannel.write(buffer);
```
### 关闭 SocketChannel 通道
在关闭 SocketChannel 传输通道前，如果传输通道用来写入数据，则建议调用一次 shutdownOutput() 终止输出方法，向对方发送一个输出的结束标志（-1）。然后调用 socketChannel.close() 方法，关闭套接字连接。
```java
// 调用终止输出方法，向对方发送一个输出的结束标志
socketChannel.shutdownOutput();

// 关闭套接字连接
IOUtil.closeQuietly(socketChannel);
```
### 通过 SocketChannel 发送文件的实践案例
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
## DatagramChannel 数据报通道
在 Java 中使用UDP协议传输数据，比 TCP 协议更加简单。和 Socket 套接字的 TCP 传输协议不同，UDP协议不是面向连接的协议。使用UDP协议时，只要知道服务器的IP和端口，就可以直接向对方发送数据。
在 Java NIO 中，使用 DatagramChannel 数据报通道来处理UDP协议的数据传输。
### 获取 DatagramChannel 数据报通道
获取数据报通道的方式很简单，调用 DatagramChannel 类的 open 静态方法即可。然后调用 configureBlocking(false) 方法，设置成非阻塞模式。
```java
//获取 DatagramChannel 数据报通道
DatagramChannel channel = DatagramChannel.open();

//设置为非阻塞模式
datagramChannel.configureBlocking(false);
```
如果需要接受数据，还需要调用 bind 方法绑定一个数据报的监听端口
```java
//调用 bind 方法绑定一个数据报的监听端口
channel.socket().bind(new InetSocketAddress(18080));
```
### 读取 DatagramChannel 数据报通道的数据
当 DatagramChannel 通道可读时，可以从 DatagramChannel 读取数据。
和前面的 SocketChannel 读取方式不同，这里不调用read方法，而是调用 receive(ByteBufferbuf) 方法将数据从 DatagramChannel 读入，再写入到ByteBuffer缓冲区中。
```java
//创建缓冲区
ByteBuffer buf = ByteBuffer.allocate(1024);

//从 DatagramChannel 读入，再写入到 ByteBuffer 缓冲区
SocketAddress clientAddr= datagramChannel.receive(buf);
```
通道读取receive（ByteBufferbuf）方法虽然读取了数据到buf缓冲区，但是其返回值是 SocketAddress 类型，表示返回发送端的连接地址（包括IP和端口）。
### 写入 DatagramChannel 数据报通道
```java
// 把缓冲区翻转到读取模式
buffer.flip();
// 调用 send 方法，把数据发送到目标 IP+端口
dChannel.send(buffer, new InetSocketAddress("127.0.0.1",18899));
// 清空缓冲区，切换到写入模式
buffer.clear();
```
### 关闭 DatagramChannel 通道
```java
dChannel.clse();
```