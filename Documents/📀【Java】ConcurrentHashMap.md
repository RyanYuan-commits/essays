### ConcurrentHashMap 和 Hashtable 的区别？​
ConcurrentHashMap 和 Hashtable 的区别主要体现在实现线程安全的方式上不同。​
**底层数据结构**： JDK1.7 的 ConcurrentHashMap 底层采用 分段的数组+链表 实现，JDK1.8 采用的数据结构跟 HashMap1.8 的结构一样，数组+链表/红黑二叉树。Hashtable 和 JDK1.8 之前的 HashMap 的底层数据结构类似都是采用 数组+链表 的形式，数组是 HashMap 的主体，链表则是主要为了解决哈希冲突而存在的；​
**实现线程安全的方式（重要）**：​
在 JDK1.7 的时候，ConcurrentHashMap 对整个桶数组进行了分割分段(Segment，分段锁)，每一把锁只锁容器其中一部分数据，多线程访问容器里不同数据段的数据，就不会存在锁竞争，提高并发访问率。​
到了 JDK1.8 的时候，ConcurrentHashMap 已经摒弃了 Segment 的概念，而是直接用 Node 数组 + 链表 + 红黑树的数据结构来实现，并发控制使用 synchronized 和 CAS 来操作。（JDK1.6 以后 synchronized 锁做了很多优化） 整个看起来就像是优化过且线程安全的 HashMap，虽然在 JDK1.8 中还能看到 Segment 的数据结构，但是已经简化了属性，只是为了兼容旧版本；​
Hashtable(同一把锁) :使用 synchronized 来保证线程安全，效率非常低下。当一个线程访问同步方法时，其他线程也访问同步方法，可能会进入阻塞或轮询状态，如使用 put 添加元素，另一个线程不能使用 put 添加元素，也不能使用 get，竞争会越来越激烈效率越低。​
### ConcurrentHashMap JDK1.7 实现的原理是什么? ​
- 首先将数据分为一段一段（这个“段”就是 Segment）的存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据时，其他段的数据也能被其他线程访问。​
- ConcurrentHashMap 是由 Segment 数组结构和 HashEntry 数组结构组成。​
- Segment 继承了 ReentrantLock,所以 Segment 是一种可重入锁，扮演锁的角色。HashEntry 用于存储键值对数据。​
- 一个 ConcurrentHashMap 里包含一个 Segment 数组，Segment 的个数一旦初始化就不能改变。 Segment 数组的大小默认是 16，也就是说默认可以同时支持 16 个线程并发写。​
- Segment 的结构和 HashMap 类似，是一种数组和链表结构，一个 Segment 包含一个 HashEntry 数组，每个 HashEntry 是一个链表结构的元素，每个 Segment 守护着一个 HashEntry 数组里的元素，当对 HashEntry 数组的数据进行修改时，必须首先获得对应的 Segment 的锁。也就是说，对同一 Segment 的并发写入会被阻塞，不同 Segment 的写入是可以并发执行的。
### ConcurrentHashMap JDK1.8 实现的原理是什么?
JDK1.8 ConcurrentHashMap 取消了 Segment 分段锁，采用 Node + CAS + synchronized 来保证并发安全。数据结构跟 HashMap 1.8 的结构类似，数组 + 链表/红黑二叉树。Java 8 在链表长度超过一定阈值8（同时满足容量》=64）时将链表（寻址时间复杂度为 O(N)）转换为红黑树（寻址时间复杂度为 O(log(N))）。​
Java 8 中，锁粒度更细，synchronized 只锁定当前链表或红黑二叉树的首节点，这样只要 hash 不冲突，就不会产生并发，就不会影响其他 Node 的读写，效率大幅提升。
### ConcurrentHashMap JDK1.7 的实现和 1.8 的实现有什么区别?
- 线程安全实现方式：JDK 1.7 采用 Segment 分段锁来保证安全， Segment 是继承自 ReentrantLock。JDK1.8 放弃了 Segment 分段锁的设计，采用 Node + CAS + synchronized 保证线程安全，锁粒度更细，synchronized 只锁定当前链表或红黑二叉树的首节点。​
- Hash 碰撞解决方法 : JDK 1.7 采用拉链法，JDK1.8 采用拉链法结合红黑树（链表长度超过一定阈值时，将链表转换为红黑树）。​
- 并发度：JDK 1.7 最大并发度是 Segment 的个数，默认是 16。JDK 1.8 最大并发度是 Node 数组的大小，并发度更大。
### JDK1.8 中，ConcurrentHashmap的put过程是怎样的？​
整体流程跟HashMap比较类似，大致是以下几步：​
- 如果桶数组未初始化，则初始化；​
- 如果待插入的元素所在的桶为空，则尝试把此元素直接插入到桶的第一个位置；​
- 如果正在扩容，则当前线程一起加入到扩容的过程中；​
- 如果待插入的元素所在的桶不为空且没在迁移元素，则锁住这个桶；​
- 如果当前桶中元素以链表方式存储，则在链表中寻找该元素或者插入元素；​
- 如果当前桶中元素以红黑树方式存储，则在红黑树中寻找该元素或者插入元素；​
- 如果元素存在，则覆盖旧值；​
- 如果元素不存在，整个Map的元素个数加1，并检查是否需要扩容；
### ConCurrentHashmap 的key，value是否可以为null？
不行。如果 key 或者 value 为 null 会抛出空指针异常。（原因是因为没办法解决 get 返回值为 null 时的二义性问题，即没办法确定是存储的值本身为 null，还是说值不存在）；​
注意：HashMap 允许使用 null 作为值和键。（因为 HashMap 只能单线程下使用，所以 hashmap 可以用 containsKey 来二次判断，排除二义性问题）

​

​