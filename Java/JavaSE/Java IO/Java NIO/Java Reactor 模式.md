---
type: Java
sub-type: IO
---
## 1	Reactor 模式

在传统的 OIO 模型中, 我们通常采用 Connection Per Thread 的模式来处理并发请求. 

```java
// 伪代码
while(true){
    socket = server.accept(); // 阻塞, 等待新连接
    new Thread(() -> {
        handle(socket); // 读取, 处理, 写入
    }).start();
}
```

这种模式虽然简单, 但在高并发场景下有致命缺陷: 

- 资源消耗巨大: 每来一个连接就需要创建一个线程, 当连接数成千上万时, 线程的创建, 销毁以及上下文切换会带来巨大的系统开销. 
    
- 弹性伸缩性差: 线程数是有限的, 无法无限扩展以应对海量连接. 

为了解决这个问题, 我们需要一种更高效的模式, 让更少的线程处理更多的连接. Reactor 模式应运而生, 它基于 [[Java IO 底层原理#2.4 IO Multiplexing|IO 多路复用]] 技术, 是构建高性能网络服务的基石, 其充分利用多核 CPU 和线程池, 能够处理海量并发连接.

## 2	核心组成: Reactor 与 Handler

Reactor 模式的核心思想是事件驱动, 它主要由两个角色构成: 

- Reactor: 负责监听和分发事件. 它使用 IO 多路复用机制监听多个 Channel 上的 IO 事件 . 一旦事件到达, Reactor 会将事件分发给对应的 Handler. 
    
- Handler: 负责处理具体的 IO 事件. 它与一个 SelectionKey 绑定, 执行实际的业务逻辑, 如建立连接, 读取数据, 编码解码, 计算和响应等. 

## 3	模式演进

这是最基础的 Reactor 模式. 所有的 IO 操作以及业务逻辑处理都在同一个线程中完成. 

![[单线程版本的 Reactor 反应器模式.png#pic_center|600]]

这种模式没有多线程的复杂性, 不存在线程安全问题. 并且避免了不必要的线程切换. 

但是其只有一个线程, 无法充分利用多核 CPU 的性能. 并且, 一旦某个 Handler 的业务处理发生阻塞, 整个系统都会被阻塞, 无法响应任何其他事件. 所以, 单线程 Reactor 模式在生产环境中几乎不被使用. 

---

为了解决单线程模型的缺点, 将业务处理部分放入一个独立的线程池中执行. 避免了耗时业务阻塞 IO 线程. 但是, Reactor 仍然是单线程, 在高并发下, 它需要处理所有连接的事件监听, 分发, 可能会成为新的性能瓶颈. 

---

于是就出现了主从 Reactor, 这是业界最常用, 最成熟的 Reactor 模式, Netty, Redis 等都采用了类似的模型. 它将 Reactor 角色进一步细分为 MainReactor 和 SubReactor. 在这种模式下, 即使某个 SubReactor 中的某个业务处理稍有延迟, 也只会影响该 SubReactor 负责的部分连接, 而不会影响 MainReactor 接收新连接, 也不会影响其他 SubReactor. 

- MainReactor: 通常只有一个, 独立一个线程. 它只负责一件事: 监听服务端的  ACCEPT 事件, 即处理新的客户端连接. 当接收到新连接后, 它会将这个连接注册到某个 SubReactor 上.
	
- SubReactor: 可以有多个, 每个 SubReactor 也独立运行在一个线程中. 它们负责处理已连接 Channel上的读写事件 (READ/WRITE), 并调用相应的 Handler 完成业务处理. 

## 4	代码示例

### 4.1	AcceptorHandler

AcceptorHandler 负责接收新连接, 并以轮询的方式将新连接注册到其中一个 SubReactor 上. 

```java
// AcceptorHandler.java
public class AcceptorHandler implements Runnable {
    final ServerSocketChannel ssc;
    final Selector[] workSelectors;
    private final AtomicInteger next = new AtomicInteger(0);

    public AcceptorHandler(ServerSocketChannel ssc, Selector[] workSelectors) {
        this.ssc = ssc;
        this.workSelectors = workSelectors;
    }

    @Override
    public void run() {
        try {
            SocketChannel channel = ssc.accept(); // 接收新连接
            if (channel != null) {
                System.out.println("接收到一个新连接: " + channel.getRemoteAddress());
                int index = next.getAndIncrement() % workSelectors.length; // 轮询选择一个 SubReactor
                Selector selector = workSelectors[index];
                // 将新连接注册到 SubReactor 上, 并绑定一个 Handler
                new MultiThreadEchoHandler(selector, channel);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 4.2	MultiThreadEchoHandler

`MultiThreadEchoHandler` 负责处理读写事件, 并将耗时的业务逻辑提交到独立的业务线程池中执行, 从而彻底将 IO 线程与业务线程分离. 

```java
// MultiThreadEchoHandler.java
public class MultiThreadEchoHandler implements Runnable {
    final SocketChannel sc;
    final SelectionKey sk;
    final ByteBuffer buf = ByteBuffer.allocate(1024);
    static final int RECEIVING = 0, SENDING = 1;
    int state = RECEIVING;

    // 独立的业务线程池
    static ExecutorService pool = Executors.newFixedThreadPool(4);

    public MultiThreadEchoHandler(Selector selector, SocketChannel sc) throws IOException {
        this.sc = sc;
        sc.configureBlocking(false);
        sk = sc.register(selector, SelectionKey.OP_READ); // 注册 READ 事件
        sk.attach(this); // 将当前 Handler 作为附件绑定
        selector.wakeup(); // 唤醒 Selector, 使注册生效
    }

    @Override
    public void run() {
        // 将业务处理提交到线程池, 实现 IO 线程与业务线程分离
        pool.execute(this::asyncRun);
    }

    // 异步执行的业务逻辑
    public synchronized void asyncRun() {
        try {
            if (state == SENDING) {
                sc.write(buf);
                buf.clear();
                sk.interestOps(SelectionKey.OP_READ);
                state = RECEIVING;
            } else if (state == RECEIVING) {
                int length = sc.read(buf);
                if (length > 0) {
                    System.out.println("收到消息: " + new String(buf.array(), 0, length));
                    buf.flip();
                    sk.interestOps(SelectionKey.OP_WRITE);
                    state = SENDING;
                }
            }
        } catch (IOException ex) {
            ex.printStackTrace();
            sk.cancel(); // 出现异常时取消 Key
        }
    }
}
```