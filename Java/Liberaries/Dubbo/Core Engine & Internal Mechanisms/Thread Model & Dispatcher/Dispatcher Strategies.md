---
type: Java
sub-type: Dubbo
---
像 Dubbo 这样高性能的 RPC 框架, 核心的矛盾在于 "如何高效, 安全的协调网络 I/O 操作与业务逻辑处理". **IO 操作**通常由专门的 IO 线程处理, 它们追求极致的响应速度, 不应该被任何的耗时操作阻塞; **业务操作**的执行时间可能很长, 且变化不定, 如果直接在 IO 线程上执行业务操作, 一旦业务代码变慢, 整个 IO 线程就会被卡住, 导致服务端无法接收任何客户端的新请求, 从而引发雪崩效应.

因此, Dubbo 需要一个分发器, 来将请求从 IO 线程转发到专门处理业务逻辑的线程池, Dispatcher Strategies 就是这个 Dispatcher 的不同工作模式.

## 1	Dispatcher Strategies

当 IO 线程解码出一个完整的 Dubbo 请求后, 它会调用 `Dispatcher` 的 `dispatch` 方法, 不同的执行策略有: 

### 1.1	All Dispatcher

![[All Dispatcher Thread Module.png|900]]

`Dispatcher` 将接收到的任何事件, 包括业务请求, 心跳, 连接, 断开等, 都封装成一个 `Runnable` 实例, 并提交到 Business ThreadPool 中; IO 线程提交后立刻返回, 无需等待执行结果; 适用于绝大多数的场景, 确保 IO 线程绝对不会被业务逻辑阻塞, 系统吞吐量稳定.

### 1.2	Direct Dispatcher
![[Direct Dispatcher.png|900]]

不执行任何分发，直接在当前的 IO 线程中执行业务逻辑; 只建议在业务逻辑绝对非阻塞时使用.

### 1.3	Message Only Dispatcher
![[Message Only Dispatcher.png|900]]

如果事件类型是 Request 或 Response，则将其封装成任务并提交到业务线程池; 适合希望将业务处理逻辑和连接管理逻辑完全隔离的场景下使用.

### 1.4	Execution Dispatcher
![[Execution Dispatcher.png|900]]

只针对 Request 类型进行分发; 如果在 Message Only Dispatcher 的基础上, 响应处理逻辑较为简单, 可以使用.

### 1.5	Connection Ordered Dispatcher
![[Connection Ordered Dispatcher.png|900]]

在 `all` 策略的基础上, 为每个 `Connection` 建立一个独立的串行队列, 保证同一个客户端的任务顺序执行.