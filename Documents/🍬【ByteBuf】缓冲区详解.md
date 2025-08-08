## 1 ByteBuf 的优势

与 Java NIO 的 `ByteBuffer` 相比, ByteBuf的优势如下: 

- Pooling (池化机制), 这点减少了内存的分配和释放, 减少了GC, 提升了效率
- 零复制机制(如复合缓冲区类型), 这点减少了内存复制
- 不需要调用flip方法去切换读/写模式
- 可扩展性好
- 可以自定义缓冲区类型
- 读取和写入索引分开
- 方法的链式调用
- 可以进行引用计数, 方便重复使用

## 2 ByteBuf 的四个组成部分

`ByteBuf` 内部是一个字节数组, `ByteBuf` 将这个数组从逻辑上分为了这几个部分: 

![[ByteBuf 的四个组成部分.png|800]]

第一个部分是已用字节, 表示已经使用完的废弃的无效字节；

第二部分是可读字节, 这部分数据是 `ByteBuf` 保存的有效数据, 从 ByteBuf 中读取的数据都来自这一部分；

第三部分是可写字节, 写入到 `ByteBuf` 的数据都会写到这一部分中；

第四部分是可扩容字节, 表示 `ByteBuf` 最多还能扩容多少. 
## 3 ByteBuf 的三个重要属性

这些属性定义在 ByteBuf 的抽象类 —— AbstractByteBuf 中: 

```java
int readerIndex;  // 读指针
int writerIndex;  // 写指针
private int maxCapacity; // 最大容量
```


![[ByteBuf 中对三个重要属性.png|700]]

`readIndex`: 读取的起始位置, 当 `readIndex = writeIndex` 的时候, ByteBuf 就不可读了

`writeIndex`: 写入的起始位置, 当 `writeIndex = capacity` 的时候, 就说明不可写入了, `capacity` 是一个成员方法, 表示 `ByteBuf` 可写的容量, 而它的值不一定就是 `maxCapacity`

`maxCapacity`: 表示 `ByteBuf` 可以扩容的最大容量, 当写入数据的时候, 如果容量不足, 可以进行扩容, 扩容的最大限度就是 `maxCapacity`. 

## 4 ByteBuf 的三组方法

### 4.1 容量系列方法

`capacity`: 表示 ByteBuf 的容量, 它的值是以下三部分之和: 废弃的字节数、可读字节数和可写字节数. 

`maxCapacity`: 表示 ByteBuf 最大能够容纳的最大字节数. 当向 ByteBuf 中写数据的时候, 如果发现容量不足, 则进行扩容, 直到扩容到 maxCapacity 设定的上限. 

### 4.2 写入系列方法

`isWritable` : 表示 `ByteBuf` 是否可写. 如果 `capacity` 容量大于 `writerIndex` 指针的位置, 则表示可写, 否则为不可写. 
- 注意: 如果 `isWritable` 返回 `false`, 并不代表不能再往 `ByteBuf` 中写数据了. 如果 Netty 发现往 `ByteBuf` 中写数据写不进去的话, 会自动扩容 `ByteBuf`. 

`writableBytes` : 取得可写入的字节数, 它的值等于容量 capacity 减去 writerIndex. 

`maxWritableBytes` : 取得最大的可写字节数, 它的值等于最大容量 `maxCapacity` 减去 `writerIndex`. 

`writeBytes(byte[] src)` : 把入参 `src` 字节数组中的数据全部写到 `ByteBuf`. ==这是最为常用的一个方法. ==

`writeTYPE(TYPE value) `: 写入基础数据类型的数据. `TYPE` 表示基础数据类型, 包含了 8 大基础数据类型. 具体如下: `writeByte`、 `writeBoolean`、`writeChar`、`writeShort`、`writeInt`、`writeLong`、`writeFloat`、`writeDouble`

`setTYPE(int index, TYPE value) `: 基础数据类型的设置, **不改变 writerIndex 指针值**, 包含了 8 大基础数据类型的设置. 具体如下: `setByte`、 `setBoolean`、`setChar`、`setShort`、`setInt`、`setLong`、`setFloat`、`setDouble`. 

`markWriterIndex` 与 `resetWriterIndex`: 这两个方法一起介绍. 前一个方法表示把当前的写指针 `writerIndex` 属性的值保存在 `markedWriterIndex` 标记属性中；后一个方法表示把之前保存的 `markedWriterIndex` 的值恢复到写指针 `writerIndex` 属性中. 这两个方法都用到了标记属性 `markedWriterIndex`, 相当于一个写指针的暂存属性. 

### 4.3 读取系列方法
isReadable( ) : 返回ByteBuf是否可读. 如果 writerIndex 指针的值大于 readerIndex 指针的值, 则表示可读, 否则为不可读. 

`readableBytes`: 返回表示 `ByteBuf` 当前可读取的字节数, 它的值等于 `writerIndex` 减去 `readerIndex`. 

`readBytes(byte[] dst)`: 将数据从 `ByteBuf` 读取到 `dst` 目标字节数组中, 这里 `dst` 字节数组的大小, 通常等于 `readableBytes` 可读字节数. 这个方法也是最为常用的一个方法之一. 

`readType`: 读取基础数据类型, 可以读取 8大基础数据类型. 具体如下: `readByte`、`readBoolean`、`readChar`、`readShort`、`readInt`、`readLong`、`readFloat`、`readDouble` . 

`getTYPE`: 读取基础数据类型, 并且不改变 readerIndex 读指针的值. 具体如下: `getByte`、 `getBoolean`、`getChar`、`getShort`、`getInt`、`getLong`、`getFloat`、`getDouble`. 

`markReaderIndex` 与 `resetReaderIndex` : 前一个方法表示把当前的读指针 `readerIndex` 保存在 `markedReaderIndex` 属性中. 后一个方法表示把保存在 `markedReaderIndex` 属性的值恢复到读指针 `readerIndex` 中. 
- `markedReaderIndex` 属性定义在 `AbstractByteBuf` 抽象基类中, 是一个标记属性, 相当于一个读指针的暂存属性. 
## 5 ByteBuf 的引用与销毁
### 5.1 ByteBuf 的引用策略简介

JVM 中使用“计数器” (一种GC算法) 来标记对象是否“不可达”进而收回, Netty也使用了这种手段来对ByteBuf 的引用进行计数, Netty 的 ByteBuf 的内存回收工作是通过引用计数的方式管理的. 
Netty 之所以采用“计数器”来追踪 ByteBuf 的生命周期, 

- 一是能对 Pooled ByteBuf 的支持, 
- 二是能够尽快地“发现”那些可以回收的 ByteBuf (非 Pooled) , 以便提升 ByteBuf 的分配和销毁的效率. 

ByteBuf 引用计数的大致规则如下: 

- 在默认情况下, 当创建完一个 ByteBuf 时, 它的引用为 1；
- 每次调用 retain 方法, 它的引用就加 1；
- 每次调用 release 方法, 就是将引用计数减 1；
- 如果引用为 0, 再次访问这个 ByteBuf 对象, 将会抛出异常；如果引用为 0, 表示这个 ByteBuf 没有哪个进程引用它, 它占用的内存需要回收. 

```java
public class ByteBufCite {  
    public static void main(String[] args) {  
        ByteBuf buffer = ByteBufAllocator.DEFAULT.buffer;  
        System.out.println("创建后: " + buffer.refCnt);  
  
        buffer.retain;  
        System.out.println("retain 后: " + buffer.refCnt);  
  
        buffer.release;  
        System.out.println("release 后: " + buffer.refCnt);  
  
        buffer.release;  
        System.out.println("release 后: " + buffer.refCnt);  
  
        // buffer.retain; 执行后会报错  
        // buffer.release; 执行后会报错  
    }  
}
```

输出结果: 

```
创建后: 1
retain 后: 2
release 后: 1
release 后: 0
```

当引用次数归零之后, 不管是 release 方法或者是 retain 方法都无法再继续执行, 也就是说, 在 Netty 张, ==引用计数为 0 的缓冲区是不能再被使用的==. 
### 5.2 ByteBuf 的销毁策略

当 ByteBuf 的引用计数已经为 0, Netty 会进行 ByteBuf 的回收. 分为两种场景: 

- 如果属于Pooled池化的 ByteBuf 内存, 回收方法是: 放入可以重新分配的 ByteBuf 池子, 等待下一次分配；
- Unpooled 未池化的 ByteBuf 缓冲区, 需要细分为两种情况: 如果是堆 (Heap) 结构缓冲, 会被 JVM 的垃圾回收机制回收；如果是 Direct 直接内存的类型, 则会调用本地方法释放外部内存 (unsafe.freeMemory) . 

除了通过ByteBuf成员方法 retain 和 release 管理引用计数之外, Netty还提供了一组件用于增加和减少引用计数的通用静态方法: 

```java
ReferenceCountUtil.retain(Object) // 增加一次缓冲区引用计数的静态方法, 从而防止该缓冲区被释放；
ReferenceCountUtil.release(Object) // 减少一次缓冲区引用计数的静态方法, 如果引用计数为 0, 缓冲区将被释放. 
```
### 5.3 ByteBuf 的自动创建与自动释放

#### ByteBuf 的自动创建

```java
@Override
public final void read {
    final ChannelConfig config = config;
    final ChannelPipeline pipeline = pipeline;
    // 获取通道的缓冲区分配器
    final ByteBufAllocator allocator = config.getAllocator;
    final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle;
    allocHandle.reset(config);

    ByteBuf byteBuf = null;
    boolean close = false;
    try {
        do {
            byteBuf = allocHandle.allocate(allocator);
            // 读取数据到缓冲区中
            allocHandle.lastBytesRead(doReadBytes(byteBuf));
            if (allocHandle.lastBytesRead %3C= 0) {
                // nothing was read. release the buffer.
                byteBuf.release;
                byteBuf = null;
                close = allocHandle.lastBytesRead < 0;
                break;
            }
            // 发送数据到 pipeline
            pipeline.fireChannelRead(byteBuf);
            ......
        } while (allocHandle.continueReading);
        ......
    } catch (Throwable t) {
        // ......
    } finally {
        // ......
    }
}

```
#### ByteBuf 的自动释放: TailContext 自动释放

在 `tailcontext` 中对 `channelRead` 方法中会调用 `onUnhandledInboundMessage` 方法, 并最终执行 `ReferenceCountUtil.release(msg)`;  来执行一次释放. 

```java
@Override  
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {  
    onUnhandledInboundMessage(msg);  
}

protected void onUnhandledInboundMessage(Object msg) {  
    try {  
        logger.debug(  
                "Discarded inbound message {} that reached at the tail of the pipeline. " +  
                        "Please check your pipeline configuration.", msg);  
    } finally {  
        ReferenceCountUtil.release(msg);  
    }  
}
```

#### ButeBuf 的自动释放: SimpleChannelInboundHandler

以入站读数据为例, `Handler` 业务处理器可以继承自 `SimpleChannelInboundHandler` 基类, 此时必须将业务处理代码, 移动到重写的 `channelRead0(ctx, msg)` 方法中. 

`SimpleChannelInboundHandle` 类的入站处理方法 (如 `channelRead` 等) , 会在调用完实际的 `channelRead0` 方法后, 帮忙释放 `ByteBuf` 实例. 

>[!info] io.netty.channel.SimpleChannelInboundHandler#channelRead
```java
@Override  
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {  
    boolean release = true;  
    try {  
        if (acceptInboundMessage(msg)) {  
            @SuppressWarnings("unchecked")  
            I imsg = (I) msg;  
            // 调用实际的处理方法, 子类提供实现
            channelRead0(ctx, imsg);  
        } else {  
            release = false;  
            ctx.fireChannelRead(msg);  
        }  
    } finally {  
        if (autoRelease && release) {  
            // 协助释放一次
            ReferenceCountUtil.release(msg);  
        }  
    }  
}
```
#### ByteBuf 的自动释放: 出站处理的自动释放

数据写出的时候最终会调用 `Channel` 实现的 `doWrite` 方法, 发送完成后这个犯法就会减少 `ByteBuf` 缓冲区的引用次数: 

```java
@Override  
protected void doWrite(ChannelOutboundBuffer in) throws Exception {  
    int writeSpinCount = -1;  
    boolean setOpWrite = false;  
    for (;;) {  
        Object msg = in.current;  
        if (msg == null) {  
            clearOpWrite;  
            return;  
        }  
        if (msg instanceof ByteBuf) {  
            ByteBuf buf = (ByteBuf) msg;  
            int readableBytes = buf.readableBytes;  
            if (readableBytes == 0) { 
                // 方法中包含释放 msg 的代码 
                in.remove;  
                continue;  
            }  
            boolean done = false;  
            long flushedAmount = 0;  
            if (writeSpinCount == -1) {  
                writeSpinCount = config.getWriteSpinCount;  
            }  
            for (int i = writeSpinCount - 1; i >= 0; i --) {  
                // 发送缓冲区的字节数据到 Java NIo 通道
                int localFlushedAmount = doWriteBytes(buf);  
                ......
        } else if (msg instanceof FileRegion) {  
            ......
        } else {  
            // Should not reach here.  
            throw new Error;  
        }  
    }  
    incompleteWrite(setOpWrite);  
}

@Override  
protected int doWriteBytes(ByteBuf buf) throws Exception {  
    final int expectedWrittenBytes = buf.readableBytes;  
    return buf.readBytes(javaChannel, expectedWrittenBytes);  
}
```
#### ByteBuf 的释放原则

由 `HeadContext` 传递过来的原始 `ByteBuf`, 如果一路传播到 `TailContext`, 这时无须手动释放, 由 `TailContext` 自动释放. 比如, 可以调用 `ctx.fireChannelRead(msg)` 向后传递, 一路将原始 `ByteBuf` 传播到尾. 

- 在流水线处理的过程中, 如果 `ByteBuf` 终止传播, 不能向后传播到 `TailContext`, 那么, 必须调用 `release` 手动释放, 或者通过继承 `SimpleChannelInboundHandler` 实现自动释放. 
- 在流水线处理的过程中, 如果某个处理器将原始 `ByteBuf` 转换为其它类型的 Java 对象, 这时 `ByteBuf` 就没用了, 必须调用 `release` 手动释放. 
- 在流水线处理的过程中, 如果原始的 `ByteBuf` 中途被替换成别的 `ByteBuf`, 那么原始 `ByteBuf` 需要手动释放. 
- 在流水线处理的过程中, 如果发生异常, 导致 `ByteBuf` 没有成功传递到下一个 `ChannelHandler`, 从而最终没有到达 `TailContext`, 必须调用 `release` 手动释放. 

出站处理时, `ByteBuf` 释放的原则, 大致如下: 

- 默认情况下, 出站的消息时普通 Java 对象, 最终都会转为 `ByteBuf` 输出, 一直向前传, 由 `HeadContext` 完成自动释放. 而普通 Java 对象 由 JVM 垃圾回收期负责回收. 
- 在流水线的出站传播过程中, 如果某个 `ByteBuf` 被终止传播, 从而最终没有传播到流水线头部 `HeadContext`, 那么, 必须调用 `release` 手动释放. 

## 6 ByteBuffer 的池化与非池化

### 6.1 关于 ByteBufAllocator

Netty 通过 `ByteBufAllocator` 分配器来创建缓冲区和分配内存空间. Netty 提供了两种分配器实现: `PoolByteBufAllocator` 和 `UnpooledByteBufAllocator`. 

- `PoolByteBufAllocator` (池化的 `ByteBuf` 分配器) 将 `ByteBuf` 实例放入池中, 提高了性能, 将内存碎片减少到最小；池化分配器采用了类 jemalloc 的高效内存分配的策略, 该策略被好几种现代操作系统所采用. 
- `UnpooledByteBufAllocator` 是普通的未池化 `ByteBuf` 分配器, 它没有把 `ByteBuf` 放入池中, 每次被调用时, 返回一个新的 `ByteBuf` 实例；使用完之后, 通过 Java 的垃圾回收机制回收或者直接释放 (对于直接内存而言) . 
在通信程序的数据传输过程中, `Buffer` 缓冲区实例会被频繁创建、使用、释放, 而频繁创建对象、内存分配、释放内存, 这样导致系统的开销大、性能低, 如何提升性能、提高 Buffer 实例的使用率呢？
池化 `ByteBuf` 是一种非常有效的方式, 所以, ==Netty 默认使用的分配器为 PoolByteBufAllocator==. 
在 Netty 中, 默认的分配器为`ByteBufAllocator.DEFAULT`, 该默认的分配器可以通过 JVM 参数或者系统选项 (System Property) `io.netty.allocator.type` 进行配置, 配置时使用字符串值: "unpooled", "pooled". 

```
-Dio.netty.allocator.type={ upooled | pooled }
```

不同的 Netty 版本, 对于分配器的默认使用策略是不一样的. 在 Netty 4.0 版本中, 默认的分配器为 UnpooledByteBufAllocator 非池化内存分配器. 

而在 Netty 4.1 版本中, 默认的分配器为 PooledByteBufAllocator 池化内存分配器, 初始化代码在 ByteBufUtil 类中的静态代码中, 如下: 

```java
static {  
    String allocType = SystemPropertyUtil.get( "io.netty.allocator.type", PlatformDependent.isAndroid ? "unpooled" : "pooled");  
    allocType = allocType.toLowerCase(Locale.US).trim;  
  
    ByteBufAllocator alloc;  
    if ("unpooled".equals(allocType)) {  
        alloc = UnpooledByteBufAllocator.DEFAULT;  
        logger.debug("-Dio.netty.allocator.type: {}", allocType);  
    } else if ("pooled".equals(allocType)) {  
        alloc = PooledByteBufAllocator.DEFAULT;  
        logger.debug("-Dio.netty.allocator.type: {}", allocType);  
    } else {  
        alloc = PooledByteBufAllocator.DEFAULT;  
        logger.debug("-Dio.netty.allocator.type: pooled (unknown: {})", allocType);  
    }  
  
    DEFAULT_ALLOCATOR = alloc;  
  
    THREAD_LOCAL_BUFFER_SIZE = SystemPropertyUtil.getInt("io.netty.threadLocalDirectBufferSize", 64 * 1024);  
    logger.debug("-Dio.netty.threadLocalDirectBufferSize: {}", THREAD_LOCAL_BUFFER_SIZE);  
  
    MAX_CHAR_BUFFER_SIZE = SystemPropertyUtil.getInt("io.netty.maxThreadLocalCharBufferSize", 16 * 1024);  
    logger.debug("-Dio.netty.maxThreadLocalCharBufferSize: {}", MAX_CHAR_BUFFER_SIZE);  
}
```

现在 PooledByteBufAllocator 已经广泛使用了一段时间, 并且有了增强的缓冲区泄漏追踪机制. 因此, 也可以在 Netty 应用引导类 Bootstrap 装配的时候, 将 PooledByteBufAllocator 设置为默认的分配器. 

```java
ServerBootstrap b = new ServerBootstrap
b.option(ChannelOption.SO_KEEPALIVE, true);
// 设置父通道的缓冲区分配器
b.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
// 设置子通道的缓冲区分配器
b.childOption(ChannelOption.ALLOCATOR,PooledByteBufAllocator.DEFAULT);
```
### 6.2 创建 ByteBuf 的方法

```java
ByteBuf byteBufCreate {  
    ByteBuf buffer = null;  
      
    // 创建一个 9-100 的可变缓冲区  
    buffer = ByteBufAllocator.DEFAULT.buffer(9, 100);  
      
    // 初始容量为 256 最大容量为 Integer.MAX_VALUE 的缓冲区  
    buffer = ByteBufAllocator.DEFAULT.buffer;  
  
    // 非池化分配器, 分配 Java 堆结构内存缓冲区  
    buffer = UnpooledByteBufAllocator.DEFAULT.heapBuffer;  
  
    // 池化分配器, 分配由操作系统管理的直接内存缓冲区  
    buffer = PooledByteBufAllocator.DEFAULT.directBuffer;  
      
    return buffer;  
}
```
### 6.3 ByteBuf 缓冲区的类型

| 类型               | 说明                                                       | 优点                                  | 缺点                                                      |
| ---------------- | -------------------------------------------------------- | ----------------------------------- | ------------------------------------------------------- |
| Heap ByteBuf     | 内部数据为一个 Java 数组, 存储在 JVM 的堆空间中, 可以通过 hasArray 方法来判断是不是堆缓冲区 | 能提供快速的分配和释放                         | 写入底层传输通道之前, 需要一次复制, 将数据写入到直接缓冲区                           |
| Direct ByteBuf   | 内部数据存储在操作系统中对物理内存                                        | 能获取超过 JVM 内存大小限制的内存空间, 写入传输通道比对缓冲区更快 | 释放和分配空间空间昂贵 (使用了操作提供的方法) , 在 Java 中读取数据的时候, <br>需要复制一份到堆区上.  |
| Composite Buffer | 多个缓冲区的组合表示                                               | 方便一次操作多个缓冲区实例                       |                                                         |

上面三种缓冲区的类型, 无论哪一种, 都可以通过池化 (Pooled) 、非池化 (Unpooled) 两种分配器来创建和分配内存空间. 

下面对 Direct Memory (直接内存) 进行简单的介绍: 

- Direct Memory 不属于 Java 堆内存, 所分配的内存其实是调用操作系统 `malloc` 函数来获得的；由 Netty 的本地内存堆 Native 堆进行管理. 
- Direct Memory 容量可通过 `-XX:MaxDirectMemorySize` 来指定, 如果不指定, 则默认与 Java 堆的最大值 (-Xmx指定) 一样. 注意: 并不是强制要求, 有的 JVM 默认 Direct Memory与 -Xmx 值无直接关系. 
- Direct Memory 的使用避免了 Java 堆和 Native 堆之间来回复制数据. 在某些应用场景中提高了性能. 
- 在需要频繁创建缓冲区的场合, 由于创建和销毁 Direct Buffer (直接缓冲区) 的代价比较高昂, 因此不宜使用 Direct Buffer. 也就是说, Direct Buffer 尽量在池化分配器中分配和回收. 如果能将 Direct Buffer 进行复用, 在读写频繁的情况下, 就可以大幅度改善性能. 
- 对 Direct Buffer 的读写比 Heap Buffer 快, 但是它的创建和销毁比普通 Heap Buffer 慢. 
- 在 Java 的垃圾回收机制回收 Java 堆时, Netty 框架也会释放不再使用的 Direct Buffer 缓冲区, 因为它的内存为堆外内存, 所以清理的工作不会为 Java 虚拟机 (JVM) 带来压力. 