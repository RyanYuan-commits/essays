# 基本介绍
首先，回顾一个前面介绍的基础知识，什么是IO多路复用模型？
IO多路复用指的是一个进程/线程可以同时监视多个文件描述符（含socket连接），一旦其中的一个或者多个文件描述符可读或者可写，该监听进程/线程能够进行IO事件的查询。
在Java应用层面，如何实现对多个文件描述符的监视呢？
需要用到一个非常重要的Java NIO组件——Selector 选择器。Selector 选择器可以理解为一个IO事件的监听与查询器。通过选择器，一个线程可以查询多个通道的IO事件的就绪状态。
## IO 事件
在介绍Selector选择器之前，首先介绍一下这个前置的概念：IO事件。
什么是IO事件呢？表示通道某种IO操作已经就绪、或者说已经做好了准备。例如，如果一个新Channel链接建立成功了，就会在Server Socket Channel上发生一个IO事件，代表一个新连接一个准备好，这个IO事件叫做“接收就绪”事件。
再例如，一个Channel通道如果有数据可读，就会发生一个IO事件，代表该连接数据已经准备好，这个IO事件叫做 “读就绪”事件。
Java NIO将NIO事件进行了简化，只定义了四个事件，这四种事件用SelectionKey的四个常量来表示：
- SelectionKey.OP_CONNECT
- SelectionKey.OP_ACCEPT
- SelectionKey.OP_READ
- SelectionKey.OP_WRITE
这里的IO事件不是对通道的IO操作，而是通道处于某个IO操作的就绪状态，**表示通道具备执行某个 IO 操作的条件**。
	比方说某个 SocketChannel 传输通道，如果完成了和对端的三次握手过程，则会发生“连接就绪”（OP_CONNECT）的事件。
	再比方说某个ServerSocketChannel服务器连接监听通道，在监听到一个新连接的到来时，则会发生“接收就绪”（OP_ACCEPT）的事件。
	还比方说，一个 SocketChannel 通道有数据可读，则会发生“读就绪”（OP_READ）事件；
	一个等待写入数据的SocketChannel通道，会发生写就绪（OP_WRITE）事件
## Selector 简介
了解了IO事件之后，再回头来看Selector选择器。
Selector的本质，就是去查询这些IO就绪事件，所以，它的名称就叫做 Selector查询者。
从编程实现维度来说，IO多路复用编程的第一步，是把通道注册到选择器中，第二步则是通过选择器所提供的事件查询（select）方法，这些注册的通道是否有已经就绪的IO事件（例如可读、可写、网络连接完成等）。
由于一个选择器只需要一个线程进行监控，所以，我们可以很简单地使用一个线程，通过选择器去管理多个连接通道。
![[Selector 组件只需要单线程监控.png]]
与OIO相比，NIO使用选择器的最大优势：系统开销小，系统不必为每一个网络连接（文件描述符）创建进程/线程，从而大大减小了系统的开销。
总之，通过Java NIO可以达到一个线程负责多个连接通道的IO处理，这是非常高效的。
这种高效，恰恰就来自于Java的选择器组件Selector以及其底层的操作系统IO多路复用技术的支持。
## SelectableChannel 可选择通道
**并不是所有的通道，都是可以被选择器监控或选择的。**
比方说，FileChannel文件通道就不能被选择器复用。判断一个通道能否被选择器监控或选择，有一个前提：判断它是否继承 了抽象类SelectableChannel（可选择通道），如果是则可以被选择，否则不能。
SelectableChannel 提供了实现通道的可选择性所需要的公共方法。Java NIO 中所有网络链接 Socket 套接字通道，都继承了 SelectableChannel 类，都是可选择的。
而 FileChannel 文件通道，并没有继承 SelectableChannel，因此不是可选择通道。
## SelectionKey 选择键
### 什么是 SelectionKey？
通道和选择器的监控关系，本质是一种多对一的关联关系。
这种关联关系，非常类似于数据库两个主表之间的关联关系，通道（Channel）和选择器（Selector）类似于数据库的主表，而选择键（SelectionKey）就类似于关联表。具体体现在：Selector 并不直接去管理 Channel，而是直接管理 SelectionKey，通过 SelectionKey 与 Channel 发生关系。
而Java NIO 源码中规定了，一个 Channel 最多能向 Selector 注册一次，注册之后就形成了唯一的 SelectionKey，然后被 Selector 管理起来。Selector 有一个核心成员 keys，专门用于管理注册上来的SelectionKey，Channel 注册到 Selector 后所创建的那一个唯一的 SelectionKey，添加在这个 keys 成员中，这是一个 HashSet 类型的集合。除了成员 keys 之外，**Selector 还有一个核心成员 selectedKeys，用于存放已经发生了IO事件的 SelectionKey**。
两核心成员 keys、selectedKeys 定义在Selector的抽象实现类 SelectorImpl 中，代码如下
```java
public abstract class SelectorImpl extends AbstractSelector  {  
    // The set of keys with data ready for an operation  
    protected Set<SelectionKey> selectedKeys;  
    
    // The set of keys registered with this Selector  
    protected HashSet<SelectionKey> keys;
}
```
### SelectionKey 和 IO 事件之间的关系
实际上，**SelectionKey 是 IO 事件的记录者（或存储者）**，SelectionKey 有两个核心成员，存储着自己关联的Channel上的感兴趣IO事件和已经发生的IO事件。
这两个核心成员定义在实现类 SelectionKeyImpl 中，代码如下：
```java
public class SelectionKeyImpl extends AbstractSelectionKey  {  
    final SelChImpl channel; // 关联的 Channel 实例
    public final SelectorImpl selector;   // 关联的 Selector
    private int index;  
    private volatile int interestOps;  // 在关联的 Channel 上感兴趣的 IO 实践
    private int readyOps; // 已经发生的 IO 实践，来自关联的 Channel
}
```
Channel 通道上可以发生多种IO事件，比如说读就绪事件、写就绪事件、新连接就绪事件，但是SelectionKey记录事件的成员却是一个整数类型。
这个整型通过特定比特位来代表特定的事件。具体的IO事件所占用的哪一个比特位，通过常量的方式定义在抽象类 SelectionKey 中，如下
```java
public static final int OP_READ = 1 << 0;  
public static final int OP_WRITE = 1 << 2;  
public static final int OP_CONNECT = 1 << 3;  
public static final int OP_ACCEPT = 1 << 4;
```
### Selector 与 SelectionKey 的配合
通道和选择器的监控关系注册成功后，Selector 就可以查询就绪事件。具体的查询操作，是通过调用选择器 Selector 的 select() 系列方法来完成。
通过select系列方法，选择器会通过 JNI，去进行底层操作系统的系统调用（比如select/epoll），可以不断地查询通道中所发生操作的就绪状态（或者IO事件），并且把这些发生了底层 IO 事件，转换成 Java NIO 中的 IO 事件，记录在的通道关联的SelectionKey 的 readyOps 上。除此之外，发生了 IO 事件的选择键，还会记录在 Selector 内部 selectedKeys 集合中。
简单来说，一旦在通道中发生了某些IO事件（就绪状态达成），这个事件就被记录在 SelectionKey 的 readyOps上，并且这个 SelectionKey 被记录在 Selector 内部的 selectedKeys 集合中。
当然，这里有两个前提：
	（1）通道必须在Selector注册过；
	（2）所发生的事件必须是SelectionKey上interestOps成员记录的事件
## Selector 的使用流程
**第一步：获取选择器实例**。选择器实例是通过调用静态工厂方法open()来获取的，具体如下：
```java
//调用静态工厂方法 open()来获取 Selector 实例
Selector selector = Selector.open();
```
Selector 选择器的类方法 open() 的内部，是向选择器 SPI（SelectorProvider）发出请求，通过默认的 SelectorProvider（选择器提供者）对象，获取一个新的选择器实例。
[[🛠️ Java 的 SPI 机制]]

**第二步：将通道注册到选择器实例**。要实现选择器管理通道，需要将通道注册到相应的选择器上：
```java
// 2.获取通道
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();

// 3.设置为非阻塞
serverSocketChannel.configureBlocking(false);

// 4.绑定连接
serverSocketChannel.bind(new InetSocketAddress(18899));

// 5.将通道注册到选择器上,并制定监听事件为：“接收连接”事件
serverSocketChannel.register(selector，SelectionKey.OP_ACCEPT);
```
注册到选择器的通道，必须处于非阻塞模式下，否则将抛出 IllegalBlockingModeException 异常。这意味着，FileChannel 不能与选择器一起使用，因为 FileChannel 只有阻塞模式，不能切换到非阻塞模式；而Socket套接字相关的所有通道都可以。

其次，还需要注意：一个通道，并不一定要支持所有的四种 IO 事件。
	例如服务器监听通道 ServerSocketChannel，仅仅支持 Accept（接收到新连接）IO 事件；
	而传输通道 SocketChannel 则不同，该类型通道不支持 Accept 类型的 IO 事件。
如何判断通道支持哪些事件呢？可以在注册之前，可以通过通道的 validOps() 方法，来获取该通道所有支持的IO事件集合。

**第三步：选出感兴趣的 IO 就绪事件（选择键集合）**。通过 Selector 选择器的 select() 方法，选出已经注册的、已经就绪的 IO 事件，并且保存到 SelectionKey 选择键集合中。
SelectionKey 集合保存在选择器实例内部，其元素为 SelectionKey 类型实例。调用选择器的 selectedKeys() 方法，可以取得选择键集合。
```java
//轮询，选择感兴趣的 IO 就绪事件（选择键集合）
while (selector.select() > 0) {
	Set selectedKeys = selector.selectedKeys();
	Iterator keyIterator = selectedKeys.iterator();
	while(keyIterator.hasNext()) {
		SelectionKey key = keyIterator.next();
		//根据具体的 IO 事件类型，执行对应的业务操作
		if(key.isAcceptable()) {
			// IO 事件：ServerSocketChannel 服务器监听通道有新连接
		} else if (key.isConnectable()) {
			// IO 事件：传输通道连接成功
		} else if (key.isReadable()) {
			// IO 事件：传输通道可读
		} else if (key.isWritable()) {
			// IO 事件：传输通道可写
		}
		//处理完成后，移除选择键
		keyIterator.remove();
	}
}
```
**处理完成后，需要将选择键从这个 SelectionKey 集合中移除，防止下一次循环的时候，被重复的处理**。
SelectionKey 集合不能添加元素，如果试图向 SelectionKey 选择键集合中添加元素，则将抛出 java.lang.UnsupportedOperationException 异常。
用于选择就绪的IO事件的select()方法，有多个重载的实现版本，具体如下：
	（1）select()：阻塞调用，一直到至少有一个通道发生了注册的IO事件。
	（2）select(long timeout)：和select()一样，但最长阻塞时间为timeout指定的毫秒数。
	（3）selectNow()：非阻塞，不管有没有IO事件，都会立刻返回。
select() 方法的返回值的是整数类型（int），表示发生了IO事件的数量。更准确地说，是指**从上次 select 到本次 select 期间，发生了选择器感兴趣（注册过）的 IO 事件数**。
