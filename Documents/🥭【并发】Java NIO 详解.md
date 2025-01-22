# NIO 技术的起源
## Java BIO 案例
>[!info] BioDemoHandler
```java
public class BioDemoHandler implements Runnable {  
    final Socket socket;  
    public BioDemoHandler(Socket socket) {  
        this.socket = socket;  
    }  
  
    @Override  
    public void run() {  
        boolean completed = false;  
        while (!completed) {  
            try {  
                byte[] bytes = new byte[1024];  
                int read = socket.getInputStream().read(bytes);  
                // 如果读取到结束标志  
                completed = true;  
                socket.close();  
                /* 处理业务逻辑，获取处理结果 */                
                byte[] output = new byte[1024];  
                /* 写入结果 */                
                socket.getOutputStream().write(output);  
            } catch (IOException e) {  
                throw new RuntimeException(e);  
            }  
        }  
    }  
}
```

>[!info] BioDemoServer
```java
public class BioDemoServer {  
    private static final int SERVER_PORT = 8080;  
    public static void main(String[] args)  {  
        ExecutorService executorService = Executors.newFixedThreadPool(16);  
        try (ServerSocket serverSocket = new ServerSocket(SERVER_PORT);){  
            while (true) {  
                Socket accept = serverSocket.accept();  
                executorService.execute(new BioDemoHandler(accept));  
            }  
        } catch (IOException e) {  
            throw new RuntimeException(e);  
        }  
    }  
}
```
对于每一个新的网络连接，都通过线程池分配给一个专门线程去负责 IO 处理。
每个线程都独自处理自己负责的 socket 连接的输入和输出。服务器的监听线程也是独立的，任何的 socket 连接的输入和输出处理，不会阻塞到后面新 socket 连接的监听和建立，这样，服务器的吞吐量就得到了提升。早期版本的Tomcat服务器，就是这样实现的。
但是这种方式高度依赖线程，而线程是一种很“贵“的资源，具体体现在：
1. 线程的创建和销毁成本很高，线程的创建和销毁都需要通过重量级的系统调用去完成。
2. 线程本身占用较大内存，像Java的线程的栈内存，一般至少分配512K～1M的空间，如果系统中的线程数过千，整个JVM的内存将被耗用1G。
3. 线程的切换成本是很高的。操作系统发生线程切换的时候，需要保留线程的上下文，然后执行系统调用。过多的线程频繁切换带来的后果是，可能执行线程切换的时间甚至会大于线程执行的时间，这时候带来的表现往往是系统 CPU sy 值特别高（超过20%以上)的情况，导致系统几乎陷入不可用的状态。
4. 容易造成锯齿状的系统负载。因为系统负载（System Load）是用活动线程数和等待线程数来综合计算的，一旦线程数量高但外部网络环境不是很稳定，就很容易造成大量请求同时到来，从而激活大量阻塞线程从而使系统负载压力过大。
面对数十万的连接，BIO 显然是无法实现的，但是，高并发的需求却越来越普通，随着移动端应用的兴起和各种网络游戏的盛行，百万级长连接日趋普遍，此时，必然需要一种更高效的I/O处理组件——这就是Java 的NIO编程组件。
## Java NIO 简介
在 1.4 版本之前，Java IO 类库是阻塞式 IO；从1.4版本开始，引进了新的异步 IO 库，被称为 Java New IO 类库，简称为 Java NIO。
NIO 弥补了原来面向流的OIO同步阻塞的不足，它为标准 Java 代码提供了 **高速的、面向缓冲区** 的 IO。

Java NIO类库包含以下三个核心组件：
1. Channel（通道）
2. Buffer（缓冲区）
3. Selector（选择器）

Java NIO，属于 IO 多路复用模型。只不过，Java NIO组件提供了统一的应用开发API，为大家屏蔽了底层的操作系统的差异。
### NIO vs OIO
在Java中，NIO和OIO的区别，主要体现在三个方面：
1. **OIO是面向流（Stream Oriented）的，NIO是面向缓冲区（Buffer Oriented）的。**
	在面向流的OIO操作中，IO的 read() 操作总是以流式的方式顺序地从一个流（Stream）中读取一个或多个字节，因此，我们不能随意地改变读取指针的位置，也不能前后移动流中的数据。
	而NIO中引入了Channel（通道）和Buffer（缓冲区）的概念。**面向缓冲区的读取和写入，都是与Buffer进行交互**。
	用户程序只需要从通道中读取数据到缓冲区中，或将数据从缓冲区中写入到通道中。NIO不像OIO那样是顺序操作，可以随意地读取Buffer中任意位置的数据，可以随意修改Buffer中任意位置的数据。
2. **OIO的操作是阻塞的，而NIO的操作是非阻塞的。**
	OIO的操作是阻塞的，当一个线程调用read() 或 write()时，该线程被阻塞，直到有一些数据被读取，或数据完全写入。该线程在此期间不能再干任何事情了。例如，我们调用一个read方法读取一个文件的内容，那么调用read的线程会被阻塞住，直到read操作完成。
	NIO 是非阻塞的，当我们调用read方法时，系统底层已经把数据准备好了，应用程序只需要从通道把数据复制到Buffer（缓冲区）就行；
	NIO 使用了通道和通道的 IO 多路复用技术，如果没有数据，当前线程可以去干别的事情，不需要进行阻塞等待。
3. **OIO没有选择器（Selector）概念，而NIO有选择器的概念。**
	NIO技术的实现，是基于底层的IO多路复用技术实现的，比如在Windows中需要select多路复用组件的支持，在Linux系统中需要select/poll/epoll多路复用组件的支持。所以NIO的需要底层操作系统提供支持。

# NIO 核心组件
- [[🥨【NIO】Channel]]
- [[🦞【NIO】Selector]]
- [[🍌【NIO】Buffer]]

# 通过 NIO 实现的简单 Discard 服务器
Discard 服务器的功能很简单：仅仅读取客户端通道的输入数据，读取完成后直接关闭客户端通道；并且读取到的数据直接抛弃掉（Discard）。
```java
public class DiscardServerDemo {  
  
    public static void main(String[] args) throws IOException {  
        // 1. 获取选择器  
        Selector selector = Selector.open();  
        // 2. 获取通道  
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();  
        // 3. 设置非阻塞  
        serverSocketChannel.configureBlocking(false);  
        // 4. 绑定连接  
        serverSocketChannel.bind(new InetSocketAddress(8080));  
        // 5. 注册到选择器上  
        serverSocketChannel.register(selector,  
                                    SelectionKey.OP_ACCEPT);  
        // 6. 轮训感兴趣的 IO 就绪事件  
        while (selector.select() > 0) {  
            // 7. 获取所有就绪的 SelectionKey            Set<SelectionKey> selectionKeys = selector.selectedKeys();  
            Iterator<SelectionKey> iterator = selectionKeys.iterator();  
            while (iterator.hasNext()) {  
                // 8. 判断具体是什么事件  
                SelectionKey selectionKey = iterator.next();  
                if (selectionKey.isAcceptable()) {  
                    ServerSocketChannel channel = (ServerSocketChannel) selectionKey.channel();  
                    SocketChannel sc = channel.accept();  
                    sc.configureBlocking(false);  
                    sc.register(selector, SelectionKey.OP_READ);  
                } else if (selectionKey.isReadable()) {  
                    SocketChannel channel = (SocketChannel) selectionKey.channel();  
                    ByteBuffer buf = ByteBuffer.allocate(1024);  
                    channel.read(buf);  
                    System.out.println("收到客户端数据：" + new String(buf.array()));  
                }  
                // 9. 取消选择键（已经处理过的事件，就应该被取消）  
                iterator.remove();  
            }  
        }  
    }  
  
}
```