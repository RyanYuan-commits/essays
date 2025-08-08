## 1 Pipeline 的执行顺序

### 1.1 入站的执行顺序

演示 Pipeline 入站处理流程, 构建三入站处理器.

```java
class Handlers {  
  
    static class Handler1 extends ChannelInboundHandlerAdapter {  
        @Override  
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {  
            System.out.println("Handler1");  
            super.channelRead(ctx, msg);  
        }  
    }  
  
    static class Handler2 extends ChannelInboundHandlerAdapter {  
        @Override  
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {  
            System.out.println("Handler2");  
            super.channelRead(ctx, msg);  
        }  
    }  
  
    static class Handler3 extends ChannelInboundHandlerAdapter {  
        @Override  
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {  
            System.out.println("Handler3");  
            super.channelRead(ctx, msg);  
        }  
    }  
  
}
```

```java
public class PipelineTest {  
    public static void main(String[] args) {  
        ChannelInitializer<EmbeddedChannel> channelInitializer = new ChannelInitializer<EmbeddedChannel>() {  
            @Override  
            protected void initChannel(EmbeddedChannel ch) throws Exception {  
                ch.pipeline().addLast(new Handlers.Handler1());  
                ch.pipeline().addLast(new Handlers.Handler2());  
                ch.pipeline().addLast(new Handlers.Handler3());  
            }  
        };  
        EmbeddedChannel embeddedChannel = new EmbeddedChannel(channelInitializer);  
        ByteBuf buffer = ByteBufAllocator.DEFAULT.buffer();  
        embeddedChannel.writeInbound(buffer);  
    }  
}
```

最终输出结果为：

```
Handler1
Handler2
Handler3
```

在 `Pipeline` 中, `Handers` 会被构成一个双向链表, 上面代码中通过 addLast 将数据添加到双向链表的末尾, 最终形成的双向链表就是这样的：

![[入站处理器的执行顺序.png#pic_center]]

### 1.2 出栈的执行顺序

>[!info] 将上面的代码中的入站处理器修改为出站处理器, 并重写其 write 方法. 
```java
class Handlers {  
  
    static class Handler1 extends ChannelOutboundHandlerAdapter {  
        @Override  
        public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {  
            System.out.println("Handler1");  
            super.write(ctx, msg, promise);  
        }  
    }  
  
    static class Handler2 extends ChannelOutboundHandlerAdapter {  
        @Override  
        public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {  
            System.out.println("Handler2");  
            super.write(ctx, msg, promise);  
        }  
    }  
  
    static class Handler3 extends ChannelOutboundHandlerAdapter {  
        @Override  
        public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {  
            System.out.println("Handler3");  
            super.write(ctx, msg, promise);  
        }  
    }  
  
}
```

>[!info] 执行通道的 writeOutbound 方法：
```java
public class PipelineTest {  
  
    public static void main(String[] args) {  
        ChannelInitializer<EmbeddedChannel> channelInitializer = new ChannelInitializer<EmbeddedChannel>() {  
            @Override  
            protected void initChannel(EmbeddedChannel ch) {  
                ch.pipeline().addLast(new Handlers.Handler1());  
                ch.pipeline().addLast(new Handlers.Handler2());  
                ch.pipeline().addLast(new Handlers.Handler3());  
            }  
        };  
        EmbeddedChannel embeddedChannel = new EmbeddedChannel(channelInitializer);  
        ByteBuf buffer = ByteBufAllocator.DEFAULT.buffer();  
        embeddedChannel.writeOutbound(buffer);  
    }  
  
}
```

输出结果：

```
Handler3
Handler2
Handler1
```

![[出站处理器的执行顺序.png#pic_center]]

## 2 核心类：ChannelHandlerContext

==在 Netty 的设计中 `Handler` 是无状态的, 不保存和 `Channel` 有关的信息==. Handler 的目标, 是将自己的处理逻辑做得很通用, 可以给不同的 `Channel` 使用. 与 `Handler` 有不同的是, Pipeline 是有状态的, 保存了 `Channel` 的关系. 于是乎, `Handler` 和 `Pipeline` 之间, 需要一个中间角色, 把他们联系起来. 这就是——**ChannelHandlerContext** . 

不管我们定义的是哪种类型的 Handler 业务处理器, 最终它们都是以双向链表的方式保存在流水线中. 这里流水线的节点类型, 并不是前面的 Handler 业务处理器基类, 而是其包装类型：`ChannelHandlerContext` 通道处理器上下文类. 

当 `Handler` 业务处理器被添加到流水线中时, 会为其专门创建一个通道处理器上下文 `ChannelHandlerContext` 实例, 主要封装了 `ChannelHandler` 通道处理器和 `ChannelPipeline` 通道流水线之间的关联关系. 

所以流水线 `ChannelPipeline` 中的双向链接, 实质是一个由 `ChannelHandlerContext` 组成的双向链表. 而无状态的 `Handler`, 作为 `Context` 的成员, 关联在 `ChannelHandlerContext` 中. 

>[!info] io.netty.channel.DefaultChannelPipeline#addLast
```java
@Override  
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {  
    final AbstractChannelHandlerContext newCtx;  
    synchronized (this) {  
        checkMultiplicity(handler);  
        // 构建一个 ChannelHandlerContext
        newCtx = newContext(group, filterName(name, handler), handler);  
        addLast0(newCtx);  
        // ......
    }  
    callHandlerAdded0(newCtx);  
    return this;  
}
```

![[Pipeline 示意图.png#pic_center]]

`ChannelHandlerContext` 中包含了有许多方法, 主要可以分为两类：第一类是获取上下文所关联的 Netty 组件实例, 如所关联的通道、所关联的流水线、上下文内部 Handler 业务处理器实例等；第二类是入站和出站处理方法. 

在 `Channel`、`ChannelPipeline`、`ChannelHandlerContext` 三个类中, 都存在同样的出站和入站处理方法, 这些出现在不同的类中的相同方法, 功能有何不同呢？

如果通过 `Channel` 或 `ChannelPipeline` 的实例来调用这些出站和入站处理方法, 它们就会在整条流水线中传播. 

然而, 如果是通过 `ChannelHandlerContext` 上下文调用出站和入站处理方法, 就只会从当前的节点开始, 往同类型的下一站处理器传播, 而不是在整条流水线从头至尾进行完整的传播. 

## 3 核心类：HeadContext、TileContext

实际上, 通道流水线在没有加入任何处理器之前, 装配了两个默认的处理器上下文：一个头部上下文叫做 `HeadContext`、一个尾部上下文叫做 `TailContext`. 

pipeline 的创建、初始化除了保存一些必要的属性外, 核心就在于创建了 `HeadContext` 头节点和 `TailContext` 尾节点. 每个 pipeline 中双向链表结构, 从一开始就存在了 `HeadContext` 和 `TailContext` 两个节点, 后面添加的处理器上下文节点, 都在添加在 HeadContext 实例和 `TailContext` 实例之间. 

![[Pipeline Headcontex 与 TailContext.png#pice_center]]

>[!info] DefaultChannelPipeline 的构造方法
```java
protected DefaultChannelPipeline(Channel channel) {  
    this.channel = ObjectUtil.checkNotNull(channel, "channel");  
    succeededFuture = new SucceededChannelFuture(channel, null);  
    voidPromise =  new VoidChannelPromise(channel, true);  
  
    tail = new TailContext(this);  
    head = new HeadContext(this);  
  
    head.next = tail;  
    tail.prev = head;  
}
```

### 3.1 TailContext

`TailContext` 是 `pipeline` 默认实现类 `DefaultChannelPipeline` 的内部类：

```java
// A special catch-all handler that handles both bytes and messages.  
final class TailContext extends AbstractChannelHandlerContext implements ChannelInboundHandler {  
    TailContext(DefaultChannelPipeline pipeline) {  
        super(pipeline, null, TAIL_NAME, true, false);  
        setAddComplete();  
    }  
  
    @Override  
    public ChannelHandler handler() {  
        return this;  
    }
    //... 其他出站方法
}
```

是一个入站处理器. 

### 3.2 HeadContext

流水线头部的 `HeadContext` 则比 `TailContext` 复杂得多, 既是一个出站处理器、也是一个入站处理器, 还保存了一个 `unsafe` (完成实际通道传输的类) 实例, 也就是 `HeadContext` 还需要负责最终的通道传输工作. 

`HeadContext` 也是流水线默认实现类 `DefaultChannelPipeline` 的一个内部类, 大致的代码如下：

```java
public class DefaultChannelPipeline implements ChannelPipeline {
    //内部类：头部处理器和头部上下文是同一个类
    //并且头部处理器功能负责：既是出站处理器、也是入站处理器

final class HeadContext extends AbstractChannelHandlerContext
    implements ChannelOutboundHandler, ChannelInboundHandler {

        //传输操作类实例：完成通道最终的输入、输出等操作
        //此类专供 Netty 内部使用, 应用程序不能使用, 所以取名 unsafe
        private final Unsafe unsafe;
        
        //入站处理举例：入站 (从 Channel 到 Handler)读操作
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) {
            ctx.fireChannelRead(msg);
        }

        //出站处理举例：出站 (从 Handler 到 Channel)读取传输数据
        @Override
        public void read(ChannelHandlerContext ctx) {
            unsafe.beginRead();
        }
    
        //出站处理举例：出站 (从 Handler 到 Channel)写操作
        @Override
        public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
            unsafe.write(msg, promise);
        }
         // ……省略 HeadContext 其他的处理方法
    }
}
```
## 4 核心原理：Pipeline 入站和出站的双向连接操作

```java
public class DefaultChannelPipeline implements ChannelPipeline {  
    final AbstractChannelHandlerContext head;  
    final AbstractChannelHandlerContext tail;

    @Override  
    public final ChannelFuture write(Object msg) {  
        return tail.write(msg);  // 从后向前传递
    }
    @Override  
    public final ChannelPipeline fireChannelRead(Object msg) {  
        AbstractChannelHandlerContext.invokeChannelRead(head, msg);  // 从 head 向后传递
        return this;  
    }
}
    
```

完整的出站和入站处理流转过程, 都是通过调用流水线 pipeline 实例的相应的出/入站方法开启的. 先看看入站处理的流转过程, 以流水线的入站读 read 的启动过程为例, pipeline的入站流程是从 `fireXXX(…)` 方法开始的 (XXX表具体入站操作, 入站读的操作为 `ChannelRead`), 在 `fireChannelRead` 的源码中, `pipeline` 从流水线的头节点head开始, 将入站的 msg 数据, 沿着流水线上的入站处理器, 逐个向后面传递. 如果所有的入站处理过程都没有截断流水线的处理, 则该入站数据 msg (如 `ByteBuffer` 缓冲区)将一直传递到流水线的末尾, 也就是 `TailContext` 处理器. 

pipeline 的出站流程是从流水线的尾部节点 tail 开始, 将出站的 msg 数据, 沿着流水线上的出站处理器, 逐个向前面传递的出站 msg 数据在经过所有出站处理器之后, 最终, 将一直传递到流水线的头部, 也就是 HeadContext 处理器, 并且将通过其 `unsafe` 传输实例, 将二进制数据写入到底层传输通道, 完成整个的传输处理过程. 

### 4.1 AbstractChannelHandlerContext 方法解析
#### 核心成员

入站和出站被流水线启动之后, 其核心的执行流程都在流水线链表中, 而构成链表的结构是 `ChannelHandlerContext`, 其抽象实现 `AbstractChannelHandlerContext` 的核心成员如下：

```java
abstract class AbstractChannelHandlerContext extends DefaultAttributeMap  
        implements ChannelHandlerContext, ResourceLeakHint {  
    // 指向前驱节点和后继节点的指针
    volatile AbstractChannelHandlerContext next;  
    volatile AbstractChannelHandlerContext prev;

    // 标记其是否为入站节点或出站节点, 可以同时为 true
    private final boolean inbound;  
    private final boolean outbound;  

    // 所属流水线
    private final DefaultChannelPipeline pipeline;  

    // 上下文节点名称, 在加入流水线的时候可用指定
    private final String name;  

    // 执行 Handler 中逻辑的线程, 如果没有特殊指定, 就是 Channel 的默认线程
    final EventExecutor executor;
```

我们从这个视角来看看上面提到的入站和出站的两个重要方法 —— `fireChannelRead` 和 `write`：

#### fireChannelRead 方法

```java
@Override  
public ChannelHandlerContext fireChannelRead(final Object msg) {  
    invokeChannelRead(findContextInbound(), msg);  // 找到第一个入站 Context, 开始执行
    return this;  
}

static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {  
    final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);  
    EventExecutor executor = next.executor();  
    // 检测一下是否应该是当前线程执行该方法
    if (executor.inEventLoop()) {  
        next.invokeChannelRead(m);  
    } else {  
        executor.execute(new Runnable() {  
            @Override  
            public void run() {  
                next.invokeChannelRead(m);  
            }  
        });  
    }  
}
```

最终会执行到 Handler 中定义的 read 方法：

```java
private void invokeChannelRead(Object msg) {  
    if (invokeHandler()) {  
        try {  
            ((ChannelInboundHandler) handler()).channelRead(this, msg);  
        } catch (Throwable t) {  
            notifyHandlerException(t);  
        }  
    } else {  
        fireChannelRead(msg);  
    }  
}
```

在源码中可以看到, 是直接调用了 `Handler` 的 `channelRead` 方法
也就是说, 如果在 Handler 中, 没有继续调用链表中下一个 `ChannelHandlerContext` 的 `channelRead` 方法的话, `read` 会在这个 `Handler` 中断并且结束. 

这也是为什么我们在重写方法的时候需要执行下面这个语句：

```java
super.channelRead(ctx, msg);
```

#### write 方法

```java
private void write(Object msg, boolean flush, ChannelPromise promise) {  
    // 获取第一个出站处理器
    AbstractChannelHandlerContext next = findContextOutbound();  
    final Object m = pipeline.touch(msg, next);  
    EventExecutor executor = next.executor();  
    if (executor.inEventLoop()) {  
        if (flush) {  
            next.invokeWriteAndFlush(m, promise);  
        } else {  
            next.invokeWrite(m, promise);  
        }  
    } else {  
        AbstractWriteTask task;  
        if (flush) {  
            task = WriteAndFlushTask.newInstance(next, m, promise);  
        }  else {  
            task = WriteTask.newInstance(next, m, promise);  
        }  
        safeExecute(executor, task, promise, m);  
    }  
}
```