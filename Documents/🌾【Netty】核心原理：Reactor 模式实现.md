## 1 Reactor 模式中 IO 事件的处理流程

![[Java Reactor 反应器模式中 IO 事件的处理流程.png]]

### 1.1 原始流程

第 1 步：**通道注册**。IO事件源于通道（Channel），IO是和通道（对应于底层连接而言）强相关的。一个IO事件一定属于某个通道。但是，如果要查询通道的事件，首先要将通道注册到选择器。

第 2 步：**查询事件**。在反应器模式中，一个线程会负责一个反应器（或者 SubReactor 子反应器），不断地轮询，查询选择器中的IO事件（选择键）.

第 3 步：**事件分发**。如果查询到IO事件，则分发给与IO事件有绑定关系的Handler业务处理器。

第 4 步：**完成真正的 IO 操作和业务处理**，这一步由 `Handler` 业务处理器负责。

以上 4 步，就是整个反应器模式的IO处理器流程。其中，第 1 步和第 2 步，其实是 Java NIO 的功能，反应器模式仅仅是利用了 Java NIO 的优势而已。

### 1.2 Netty Reactor

Netty 的 Reactor 反应器模式实现, 对经典的 Reator 模式进行了细微的调整: 

#### 通道注册

Netty 封装了 NIO 的 Selector 组件和 Thread 线程实例，设计了自己的 Reactor 角色，名称叫做 EventLoop（事件循环）；

并且封装了 NIO 的 Channel 组件，设计了自己的传输通道组件，名字仍然叫做 Channel，只是所处的包不同。

通道注册，指的是将 Netty 的Channel注册到 EventLoop 上，对应到底层就是 NIO 的 Channel 注册到 NIO 的 Selector 上。

#### 查询事件

在 Netty 反应器模式中，一个线程会负责一个反应器 (或者 SubReactor 子反应器），EventLoop 和 Thread 也是这种一对一的模式。

一个反应器负责一个 Selector 的查询, EventLoop 内部 Thread 不断地轮询，查询选择器 Selector 中的 IO 事件，并记录在选择键上面.

#### 事件内部分发, 数据读取和发射

这里和经典的 Reactor 模式有细微的区别: 在经典 Reactor 模式中事件分发和数据读取是分开的, Reactor 负责 IO 事件的分发, Handler 负责数据的读取;

而在 Netty 的 Reactor 模式中, 反应器 `EventLoop` 则负责了事件数据读取和分发两个操作.

具体来说, `EventLoop` 能访问到通道的 Unsafe 成员，当 IO 事件发生时, 直接通过 Unsafe 成员完成 `NIO` 底层的数据读取. `EventLoop` 读取到的数据后, 会把数据发射到 `Channel` 内部的  `Pipeline` 流水线通道.

#### 流水线传播和业务处理

数据在通道的 `Pipeline` 流水线上传播，通道的流水线由 `Handler` 构成，由 `Handler` 业务处理器负责，处理完成之后，再把结果传播或者传递到下一个 `Handler`。

为啥需要 Pipeline 流水线呢？主要是由于同一个 NIO 事件，可能会有多个业务处理，比如数据的解码、数据的校验、业务的处理，所以 Netty 通过**责任链模式**将多个业务处理器组织起来，成为一个 pipeline（流水线）。

`Pipeline` 流水线由通道负责管理，属于通道的一部分。数据可以在流水线上传播，再交给流水线上的 `Handler` 来处理。`Handler` 业务处理器放置的是具体的业务逻辑，这是 Java 工程师们需要负责开发的部分。

以上 4 步, 就是整个 Netty 的 IO 处理器流程, Netty 的 Reactor 模式, 和经典 Reactor 模式实现区别很小,  主要的区别是在第 3, 4 步.

## 2 Netty 中的 Channel 通道组件

Netty 实现了一系列的 Channel 通道组件，为了支持多种通信协议，换句话说，对于每一种通信连接协议，Netty 都实现了自己的通道。

除了 Java 的 NIO，Netty 还能提供了 Java 的面向流的 OIO (Old-IO，即传统的阻塞式IO) 的处理通道。

对应到 **不同的协议**，Netty 实现了对应的通道，每一种协议基本上都有 NIO (异步IO) 和 OIO (阻塞式IO) 两个版本：

- `NioSocketChannel`：异步非阻塞 TCP Socket 传输通道。
- `NioServerSocketChannel`：异步非阻塞 TCP Socket 服务器端监听通道。
- `NioDatagramChannel`：异步非阻塞的 UDP 传输通道。
- `NioSctpChannel`：异步非阻塞 Sctp 传输通道。
- `NioSctpServerChannel`：异步非阻塞 Sctp 服务器端监听通道。
- `OioSocketChannel`：同步阻塞式 TCP Socket 传输通道。
- `OioServerSocketChannel`：同步阻塞式 TCP Socket 服务器端监听通道。
- `OioDatagramChannel`：同步阻塞式 UDP 传输通道。
- `OioSctpChannel`：同步阻塞式 Sctp 传输通道。
- `OioSctpServerChannel`：同步阻塞式 Sctp 服务器端监听通道。
	
一般来说，服务器端编程用到最多的通信协议还是 TCP 协议，其对应的 Netty 传输通道类型为 NioSocketChannel 类，其对应的 Netty 服务器监听通道类型为 `NioServerSocketChannel`.

不论是那种通道类型，在主要的 API 和使用方式上和 `NioSocketChannel` 类基本是相同的，更多是底层的传输协议不同，而 Netty 帮大家极大的屏蔽了传输差异.

在 Netty 的 `NioSocketChannel` 内部封装了一个 Java NIO 的 `SelectableChannel` 成员，通过对这个通道的封装，对 `Netty` 中的 ``NioSocketChannel`` 通道上所有的 IO 操作，最终都会落地到 Java NIO 的 `SelectableChannel` 底层通道。

这个成员定义在其父类 `AbstractNioChannel` 中:

```java
public abstract class AbstractNioChannel extends AbstractChannel {  
    private final SelectableChannel ch;
}
```

## 3 Netty 中的 Reactor 反应器

在反应器模式中, 一个反应器 (或者 `SubReactor` 子反应器) 会由一个事件处理线程负责事件查询和分发。该线程不断进行轮询，通过 Selector 选择器不断查询注册过的 IO 事件 (选择键).如果查询到 IO 事件，则分发给 `Handler` 业务处理器.

这里为大家介绍一下 Netty 中的 Reactor 反应器组件. Netty 中的反应器组件有多个实现类，这些实现类与其 Channel 通道类型相互匹配. 对应于 `NioSocketChannel` 通道，Netty 的反应器类为 `NioEventLoop`, 也就是 NIO 事件轮询.

`NioEventLoop` 类有两个重要的成员属性: 一个是 `Thread` 线程类的成员, 一个是 Java NIO `Selector` 的成员属性.

![[NioEventLoop的继承关系和主要的成员.png|500]]

`NioEventLoop` 和前面章节讲到反应器实现，在思路上是一致的: 一个 `NioEventLoop` 拥有一个 `Thread` 线程, 负责一个 Java NIO Selector 选择器的 IO 事件轮询.

在 Netty 中, 理论上来说，一个 `EventLoop` 反应器和 `NettyChannel` 通道是一对多的关系, 一个反应器可以注册成千上万的通道.

## 4 Netty 中的 Handler 处理器

### 4.1 EventLoop 内部分发

在 Netty 中, `EventLoop` 反应器内部有一个线程负责 `Java NIO` 选择器的事件的轮询, 然后进行对应的数据分发.

注意这里和经典 `Reactor` 模式的区别：Netty 的 IO 事件分发 (`Dispatch`)，`Netty` 会在 `EventLoop` 将数据先进行一些预处理，也就是会先进行一次 **内部分发**, 比如IO读事件:

- 在 `EventLoop` 的内部, 通过 `Channel` 的 `Unsafe` 成员完成数据的读取, 将输入的数据读取到 `ByteBuf` 中.
- `EventLoop` 读取到数据之后，再将输入数据分发到通道的 `Pipeline`，此次数据分发的目标，才是 Netty 的 `Handler` 处理器.

`Handler` 处理器主要是用户定义的业务处理器和相关的编解码处理. 为了和分发的概念做区分, 有的时候, 这里这一步会被称为数据发射.

### 4.2 Handler 分类

Netty 的 Handler 处理器分为两大类：

- 第一类是 `ChannelInboundHandler` 入站处理器;
- 第二类是 `ChannelOutboundHandler` 出站处理器, 二者都继承了 `ChannelHandler` 处理器接口.

### 4.3 Handler 案例

Netty 入站处理的流程是啥呢？以底层的 Java NIO 中的 `OP_READ` 输入事件为例：

- 在通道中发生了 `OP_READ` 事件后，会被 `EventLoop` 查询到，然后分发到内部的 IO 事件处理方法
- 再通过 `Unsafe` 完成具体的 NIO 的数据读取
- 之后把读取到的输入数据发射到通道的 `Pipeline`
- 数据会在流水线上依次传播到 `ChannelInboundHandler` 入站处理器, 处理器的方法 read 将被调用. 在 `read` 方法具体实现中, 可以由业务程序处理由 `Pipeline` 传播过来的数据, 再决定是否把处理结果继续在流水线上往下一站传播.

### 4.4 入站与出站处理

#### 入站处理

Netty 中的入站处理触发的方向为：由通道触发，`ChannelInboundHandler` 入站处理器负责接收(或者执行).

Netty 中的入站处理，不仅仅是 `OP_READ` 输入事件的处理, 还包括从底层通道 (如 NIO Channel) 触发, 由 Netty 通过层层传递，调用 `ChannelInboundHandler` 入站处理器进行的其他某个处理.

#### 出站处理

Netty 中的出站处理具体指的是什么呢？指的是从 `ChannelOutboundHandler` 处理器到 `Channel` 的某次 IO 操作

例如，在应用程序完成业务处理后，可以通过 `ChannelOutboundHandler` 出站处理器将处理的结果写入底层通道。

它的最常用的一个方法就是 `write()` 方法，把数据写入到通道, 同样的, Netty 中的出站处理，不仅仅包括 `write()` 方法，还包括从 `Handler` 处理器到底层 `Channel` 的方向的其他操作。

---

Netty 出站和 Java NIO 的出站在概念上有细微的区别，Java NIO 的出站指的是 `OP_WRITE` 可写事件，以及传输维度的数据写入，而 Netty 的出站处理指的的 API 调用的方向。

所以, 两个出站处理在概念不是一个维度, Netty 的出站处理是应用层开发维度的; Java NIO 的出站是数据传输维度的.

## 5 Netty 中的 Pipeline

简单来说, `Pipeline` 构成了 Netty 中 `Channel` 和 `Handler` 之间的绑定关系.

通道和 `Handler` 处理器实例之间, 是多对多的关系: 一个通道的 IO 数据可以被多个的 Handler 实例处理; 一个 `Handler` 处理器实例也能绑定到很多的 `Channel`, 处理多个通道的IO数据.

Netty 设计了一个特殊的组件，叫做 `ChannelPipeline`(通道处理流水线），将多个 `Handler` 处理器实例串在一起，形成一条流水线.

`Channel` 通过与 `Pipeline` 的绑定, 实现了与多个 `Handler` 的绑定.

`ChannelPipeline`(通道流水线) 的默认实现，实际上被设计成一个双向链表. 所有的 Handler 处理器实例被包装成了双向链表的节点, 被加入到了 `ChannelPipeline` 中.