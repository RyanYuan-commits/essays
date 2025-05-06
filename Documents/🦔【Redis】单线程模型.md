## 1 为什么说 Redis 是单线程？
主要是指 Redis 的网络 IO 和键值对读写是通过一个线程来完成的，Redis 在处理客户端请求的时候，比如 socket 读、解析、执行、socket 写，都是由一个顺序的、串行的主线程来处理的，这就是所谓的单线程。
但是 Redis 的其他功能，比如 RDB、AOP、异步删除、集群数据同步等等，是通过额外的线程来完型的。
## 2 Redis 为什么快？
1. 基于内存实现：Redis 的所有数据都存储在内存中，运算都是内存级别，所以性能比较高。
2. 数据结构简单：Redis 都数据结构是专门设计的，这些简单的数据结构查找和操作的时间大部分的时间复杂度都是 O(1)。
3. 多路复用和非阻塞 I/O：Redis 使用 I/O 多路服用技术来监听多个 Socket 链接客户端，这样就能使用一个线程来处理多个请求，减少线程切换带来的影响。
4. 避免上下文切换：避免了上下文切换和多线程竞争，省去了多线程切换带来的时间和性能上的损耗。
## 3 Redis 为什么选择过单线程？
在官方文档的中的，How can Redis use multiple CPUs or cores? 提到：
It's not very frequent that CPU becomes your bottleneck with Redis, as usually Redis is either memory or network bound.
CPU 成为 Redis 性能瓶颈的情况非常少见，通常 Redis 的性能瓶颈通常是由内存和网络带来的。
## 4 单线程带来的删除问题
**单线程带来的删除问题**：正常情况下使用 del 指令可以很快的删除数据，而当被删除的 key 是一个非常大的对象时，例如 key 包含了成千上万个元素的 hash 集合时，那么 del 指令就会造成 Redis 主线程卡顿。