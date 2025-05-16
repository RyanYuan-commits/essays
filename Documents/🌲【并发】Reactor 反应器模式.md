>Reactor（反应器）模式是高性能网络编程在设计和架构层面的基础模式，算是基础的原理性知识。只有彻底了解反应器的原理，才能真正构建好高性能的网络应用，才能轻松地学习和掌握高并发通信服务器与框架（如Netty框架、Nginx服务器）。
### Reactor 模式为何如此重要？
#### Reactor 模式简介
Reactor反应器模式由Reactor反应器线程、Handlers处理器两大角色组成，两大角色的职责分别如下：
	（1）Reactor 反应器线程的职责：负责响应 IO 事件，并且分发到 Handlers 处理器。
	（2）Handlers 处理器的职责：非阻塞的执行业务处理逻辑。
Reactor 线程负责多路 I/O 事件的查询，然后分发到一个或者多个 Handler 处理器完成 I/O 处理，所以，Reactor 模式也叫 Dispatcher 模式。
总之，Reactor 模式和操作系统底层的 IO 多路复用模型相互结合，是编写高性能网络服务器的必备技术之一。

当然，复杂的架构和技术，都是从简单版本开始演进出来的，Reactor 反应器模式同样如此。根据前面的定义，仅仅是 Reactor 模式的最为简单的一个版本。
如果需要彻底了解反应器模式，还得从最原始的OIO编程开始讲起。
#### 多线程模式下 OIO 的致命缺陷
在 Java 的 OIO 编程中，最原始的网络服务器程序，一般是用一个 while 循环，不断地监听端口是否有新的连接。
如果有，那么就调用一个处理函数来完成传输处理，示例代码如下：
```java
while(true){
	socket = accept(); //阻塞，接收连接
	handle(socket) ; //读取数据、业务处理、写入结果
}
```
这种方法的最大问题是：如果前一个网络连接的 handle（socket）没有处理完，那么后面的新连接没法被服务端接收，于是后面的请求就会被阻塞住，这样就导致服务器的吞吐量太低。这对于服务器来说，这是一个严重的问题。
为了解决这个严重的连接阻塞问题，出现了一个极为经典模式：Connection Per Thread（一个线程处理一个连接）模式。
也就是将每一个新的网络连接都分配到一个线程，每个线程都独立处理自己负责的 Socket 连接的输入和输出。
这种方式极大的提高了服务器的吞出量，但是会消耗大量的资源，线程的反复大量创建和销毁、线程的切换都需要付出相当大的代价，在高并发的应用场景下，OIO 的缺陷是致命的。
那可不可以让一个线程去处理多个连接呢？
但是实际上这样做的意义并不大，因为传统的 OIO 是阻塞模式的，在同一个时刻，一个线程只能处理一个 Socket 的读写，前一个 Socket 操作被阻塞了，其他操作同样无法并行处理。如何解决 Connection Per Thread 模式的巨大缺陷呢？一个有效途径是：使用Reactor反应器模式。用反应器模式对线程的数量进行控制，做到一个线程处理大量的连接。它是如何做到呢？首先来看简单的版本——单线程的Reactor反应器模式。
### 单线程 Reactor 模式
总体来说，Reactor 反应器模式有点儿类似事件驱动模式。在事件驱动模式中，当有事件触发时，事件源会将事件 dispatch 分发到 handler 处理器，由处理器负责事件处理。而反应器模式中的反应器角色，类似于事件驱动模式中的 dispatcher 事件分发器角色。具体来说，在反应器模式中，有 Reactor 反应器和 Handler 处理器两个重要的组件：
	（1）Reactor反应器：负责查询IO事件，当检测到一个IO事件，将其发送给相应的Handler处理器去处理。这里的IO事件，就是NIO中选择器查询出来的通道IO事件。
	（2）Handler处理器：与IO事件（或者选择键）绑定，负责IO事件的处理。完成真正的连接建立、通道的读取、处理业务逻辑、负责将结果写出到通道等。
#### 什么是单线程 Reactor 反应器模式？
什么是单线程版本的 Reactor 反应器模式呢？简单地说，Reactor 反应器和 Handers 处理器处于一个线程中执行。
![[单线程版本的 Reactor 反应器模式.png#pic_center|700]]
基于 Java NIO 实现简单的单线程版本的反应器模式需要用到 SelectionKey 选择键的几个关键的成员方法
	**void attach(Object o)** 将对象附加到选择键：此方法可以将任何的 Java POJO 对象，作为附件添加到 SelectionKey 实例。这方法非常重要，因为单线程版本的 Reactor 反应器模式实现中，可以将 Handler 处理器实例，作为附件添加到 SelectionKey 实例。
	**Object attachment()** 从选择键获取附加对象此方法与 attach(Object o) 是配套使用的，其作用是取出之前通过 attach(Object o) 方法添加到SelectionKey选择键实例的附加对象。这个方法同样非常重要，当IO事件发生时，选择键将被select方法查询出来，可以直接将选择键的附件对象取出。
在Reactor模式实现中，通过 attachment() 方法所取出的，是之前通过 attach(Object o) 方法绑定的Handler实例，然后通过该Handler实例，完成相应的传输处理。
总之，在反应器模式中，需要进行 attach 和 attachment 结合使用：
	在选择键注册完成之后，调用 attach 方法，将 Handler 实例绑定到选择键；
	当 IO 事件发生时，调用 attachment 方法，可以从选择键取出 Handler 实例，将事件分发到 Handler 处理器中，完成业务处理。
#### 单线程 Reactor 反应器模式的缺点
单线程 Reactor 反应器模式，是基于 Java 的 NIO 实现的。相对于传统的多线程 OIO，反应器模式不再需要启动成千上万条线程，避免了线程上下文的频繁切换，服务端的效率自然是大大提升了。
在单线程反应器模式中，Reactor 反应器和 Handler 处理器都执行在同一条线程上。
这样，带来了一个问题：**当其中某个 Handler 阻塞时，会导致其他所有的 Handler 都得不到执行**。在这种场景下，被阻塞的 Handler 不仅仅负责输入和输出处理的传输处理器，还包括负责新连接监听的 AcceptorHandler 处理器，这就可能导致服务器无响应。这个是非常严重的问题。因为这个缺陷，因此单线程反应器模型在生产场景中使用得比较少。
除此之外，目前的服务器都是多核的，**单线程反应器模式模型不能充分利用多核资源**。
总之，在高性能服务器应用场景中，单线程反应器模式实际使用的很少。
### 多线程 Reactor 模式
> 既然 Reactor 反应器和 Handler 处理器，挤在一个线程会造成非常严重的性能缺陷。那么，可以使用多线程，对基础的反应器模式进行改造和演进。
#### 多线程版本 Reactor 模式的演进
多线程Reactor反应器的演进，分为两个方面：
	（1）首先是升级Reactor反应器。可以考虑引入多个Selector选择器，提升查询和分发大量通道的IO事件的能力。
	（2）其次是升级Handler处理器。既要使用多线程，又要尽可能的高效率，则可以考虑使用线程池。
总体来说，多线程版本的反应器模式，大致如下：
	（1）将负责数据传输处理的IOHandler处理器的执行，放入独立的线程池中。这样，业务处理线程与负责新连接监听的反应器线程就能相互隔离，避免服务器的连接监听受到阻塞。
	（2）如果服务器为多核的CPU，可以将反应器线程拆分为多个子反应器（SubReactor）线程；同时，引入多个选择器，并且为每一个SubReactor引入一个线程，一个线程负责一个选择器的事件轮询。这样，充分释放了系统资源的能力；也大大提升反应器管理大量连接，或者监听大量传输通道的能力。
#### 实战：多线程版本的 Reactor 反应器模式
设计要求：
	（1）引入多个选择器。
	（2）设计一个新的子反应器（SubReactor）类，一个子反应器负责查询一个选择器的查询、分发。
	（3）开启多个处理线程，一个处理线程负责执行一个子反应器（SubReactor）的。为了提升效率，这里由一个线程负责一个 SubReactor 的所有操作，避免多个线程负责一个选择器，导致需要进行线程同步，从而引发性能下降的问题。
	（4）进行IO事件的分类隔离。将新连接事件OP_ACCEPT的反应处理，和普通的读（OP_READ）事件、写（OP_WRITE）事件反应处理，进行分开隔离。这里，专门用一个 SubReactor 负责新连接事件查询和分发，防止耗时的 IO 操作导致新连接事件 OP_ACCEPT 事件查询发生延迟，这个专门的反应器也做 bossReactor；与之相对应的、负责IO事件的查询、分发的反应器，叫做 workReactor。
	（5）将IO事件的查询、分发和处理线程隔离。具体来说，就是将Handler处理器的执行，不放在Reactor绑定的线程上完成。实际上，在高并发、高性能的场景下，需要将耗时的处理与IO反应处理进行隔离，耗时的Handler处理器需要在专门的线程上完成，避免IO反应处理被阻塞。
##### Reactor
```java
public class Reactor implements Runnable {  
    final Selector selector;  
  
    public Reactor(Selector selector) {  
        this.selector = selector;  
    }  
  
    @Override  
    public void run() {  
        try {  
            while (!Thread.interrupted()) {  
                int select = selector.select(1000);  
                if (select > 0) {  
                    Iterator<SelectionKey> selectionKeys = selector.selectedKeys().iterator();  
                    while (selectionKeys.hasNext()) {  
                        SelectionKey next = selectionKeys.next();  
                        dispatch(next);  
                        selectionKeys.remove();  
                    }  
                }  
            }  
        } catch (Exception e) {  
            System.out.println(e.getMessage());  
        }  
    }  
  
    void dispatch(SelectionKey selectionKey) {  
        Runnable handler = (Runnable) selectionKey.attachment();  
        //调用之前 attach 绑定到选择键的 handler 处理器对象  
        if (handler != null) {  
            handler.run();  
        }  
    }  
  
}
```
##### MultiThreadEchoServerReactor
```java
public class MultiThreadEchoServerReactor {  
  
    static ServerSocketChannel ssc;  
  
    static AtomicInteger next = new AtomicInteger(0);  
  
    static Selector bossSelector;  
    static Selector[] workSelectors = new Selector[2];  
  
    static Reactor bossReactor;  
    static Reactor[] workReactors;  
  
    static {  
        try {  
            //初始化多个 selector 选择器  
            bossSelector = Selector.open();// 用于监听新连接事件  
            workSelectors[0] = Selector.open(); // 用于监听 read、write 事件  
            workSelectors[1] = Selector.open(); // 用于监听 read、write 事件  
            ssc = ServerSocketChannel.open();  
            InetSocketAddress address = new InetSocketAddress("localhost", 8080);  
            ssc.socket().bind(address);  
            ssc.configureBlocking(false);//非阻塞  
            //bossSelector,负责监控新连接事件, 将 serverSocket 注册到 bossSelector            SelectionKey sk = ssc.register(bossSelector, SelectionKey.OP_ACCEPT);  
            //绑定 Handler：新连接监控 handler 绑定到 SelectionKey（选择键）  
            sk.attach(new AcceptorHandler(next, ssc, workSelectors));  
            
            // bossReactor 反应器，处理新连接的 bossSelector            
            bossReactor = new Reactor(bossSelector);  
            
            // 第一个子反应器，一子反应器负责一个 worker 选择器  
            Reactor workReactor1 = new Reactor(workSelectors[0]);  
            // 第二个子反应器，一子反应器负责一个 worker 选择器  
            Reactor workReactor2 = new Reactor(workSelectors[1]);  
            workReactors = new Reactor[]{workReactor1, workReactor2};  
        } catch (IOException e) {  
            throw new RuntimeException(e);  
        }  
    }  
  
    public static void main(String[] args) throws IOException {  
        // 一子反应器对应一条线程  
        new Thread(bossReactor).start();  
        new Thread(workReactors[0]).start();  
        new Thread(workReactors[1]).start();  
        System.in.read();  
    }  
  
}
```
上面是反应器的多线程版本演进代码，总共三个选择器。
第一个选择器作为 boss，专门负责查询和分发新连接事件；
第二个、三个选择器作为worker，专门负责查询和分发IO传输事件。
上面的代码创建了三个子反应器，一个 bossReactor 负责新连接事件的反应处理（查询、分发、处理），bossReactor 和 boss 选择器进行绑定；两个workReactor负责普通IO事件的查询和分发，分别绑定一个worker选择器。
服务端的监听通道注册到boss选择器，而所有的 Socket 传输通道通过轮询策略注册到 worker 选择器，从而实现了新连接监听和 IO 读写事件监听的线程分离。
#### 实战：多线程版本 Handler 处理器
##### AcceptorHandler
```java
public class AcceptorHandler implements Runnable {  
  
    final AtomicInteger next;  
    final ServerSocketChannel ssc;  
    final Selector[] workSelectors;  
  
    public AcceptorHandler(AtomicInteger next, ServerSocketChannel ssc, Selector[] workSelectors) {  
        this.next = next;  
        this.ssc = ssc;  
        this.workSelectors = workSelectors;  
    }  
  
    @Override  
    public void run() {  
        try {  
            SocketChannel channel = ssc.accept();  
            System.out.println("接收到一个新的连接");  
            if (channel != null) {  
                int index = next.get();  
                System.out.println("选择器的编号：" + index);  
                Selector selector = workSelectors[index];  
                new MultiThreadEchoHandler(selector, channel);  
            }  
        } catch (IOException e) {  
            e.printStackTrace();  
        }  
        if (next.incrementAndGet() == workSelectors.length) {  
            next.set(0);  
        }  
    }  
}
```
##### MultiThreadEchoHandler
主要的升级是引入了一个线程池（ThreadPool），使得数据传输和业务处理的代码执行在独立的线程池中，彻底地做到IO处理以及业务处理线程和反应器IO事件轮询线程的完全隔离。
```java
public class MultiThreadEchoHandler implements Runnable {  
  
    final SocketChannel sc;  
    final SelectionKey sk;  
    final ByteBuffer buf = ByteBuffer.allocate(1024);  
    static final int RECEIVING = 0, SENDING = 1;  
    int state = RECEIVING;  
    static ExecutorService pool = Executors.newFixedThreadPool(4);  
  
    public MultiThreadEchoHandler(Selector selector, SocketChannel sc) throws IOException {  
        this.sc = sc;  
        sc.configureBlocking(false);  
        sc.setOption(StandardSocketOptions.TCP_NODELAY, true);  
        sk = sc.register(selector, 0);  
        sk.attach(this);  
        sk.interestOps(SelectionKey.OP_READ);  
        // 唤醒 Selector 使 OP_READ 生效  
        selector.wakeup();  
    }  
  
    @Override  
    public void run() {  
        //异步任务，在独立的线程池中执行  
        //提交数据传输任务到线程池  
        //使得 IO 处理不在 IO 事件轮询线程中执行，在独立的线程池中执行  
        pool.execute(new AsyncTask());  
    }  
  
    class AsyncTask implements Runnable {  
        public void run() {  
            MultiThreadEchoHandler.this.asyncRun();  
        }  
    }  
  
    // 异步任务，不在 Reactor 线程中执行  
    // 数据传输与业务处理任务，不在 IO 事件轮询线程中执行，在独立的线程池中执行  
    public synchronized void asyncRun() {  
        try {  
            if (state == SENDING) {  
                // 写入通道  
                sc.write(buf);  
                //写完后,准备开始从通道读,byteBuffer 切换成写模式  
                buf.clear();  
                // 写完后,注册 read 就绪事件  
                sk.interestOps(SelectionKey.OP_READ);  
                // 写完后,进入接收的状态  
                state = RECEIVING;  
            } else if (state == RECEIVING) {  
                // 从通道读  
                int length = 0;  
                while ((length = sc.read(buf)) > 0) {  
                    System.out.println("收到消息：" + new String(buf.array(), 0, length));  
                }  
                // 读完后，准备开始写入通道,byteBuffer 切换成读模式  
                buf.flip();  
                // 读完后，注册 write 就绪事件  
                sk.interestOps(SelectionKey.OP_WRITE);  
                // 读完后,进入发送的状态  
                state = SENDING;  
            }  
            // 处理结束了, 这里不能关闭 select key，需要重复使用  
            // sk.cancel();  
        } catch (IOException ex) {  
            ex.printStackTrace();  
        }  
    }  
  
}
```
### Reactor 模式的优点和缺点
作为高性能的IO模式，反应器模式的优点如下：
- 响应快，虽然同一反应器线程本身是同步的，但不会被单个连接的IO操作所阻塞；
- 编程相对简单，最大程度避免了复杂的多线程同步，也避免了多线程的各个进程之间切换的开销；
- 可扩展，可以方便地通过增加反应器线程的个数来充分利用CPU资源。
反应器模式的缺点如下：
- 反应器模式增加了一定的复杂性，因而有一定的门槛，并且不易于调试。
- 反应器模式依赖于操作系统底层的IO多路复用系统调用的支持，如Linux中的epoll 系统调用。如果操作系统的底层不支持IO多路复用，反应器模式不会有那么高效。
- 同一个 Handler 业务线程中，如果出现一个长时间的数据读写，会影响这个反应器中其他通道的 IO 处理。例如在大文件传输时，IO 操作就会影响其他客户端（Client）的响应时间。
  因而对于这种操作，还需要进一步对反应器模式进行改进。
