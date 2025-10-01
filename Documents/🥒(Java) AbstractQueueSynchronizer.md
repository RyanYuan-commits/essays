### 1 AQS 简介
AQS 是一个用来构建锁和同步器的框架，使用AQS能简单且高效地构造出应用广泛的大量的同步器，比如我们提到的 `ReentrantLock`，`Semaphore`，其他的诸如 `ReentrantReadWriteLock`，`SynchronousQueue`，`FutureTask` 等等皆是基于 AQS 的。当然，我们自己也能利用 AQS非常轻松容易地构造出符合我们自己需求的同步器。
#### 1.1 AQS 核心思想
AQS 核心思想是，如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并且将共享资源设置为锁定状态。如果被请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制，这个机制 AQS 是用 CLH 队列锁实现的，即将暂时获取不到锁的线程加入到队列中。
CLH (Craig,Landin,and Hagersten) 队列是一个虚拟的双向队列(虚拟的双向队列即不存在队列实例，仅存在结点之间的关联关系)。AQS 是将每条请求共享资源的线程封装成一个 CLH 锁队列的一个结点(Node)来实现锁的分配。
AQS 使用一个 int 成员变量来表示同步状态，通过内置的 FIFO 队列来完成获取资源线程的排队工作。AQS 使用 CAS 对该同步状态进行原子操作实现对其值的修改，状态信息通过procted类型的 `getState`，`setState`，`compareAndSetState` 进行操作：
```java
private volatile int state; // 共享变量，使用volatile修饰保证线程可见性

protected final int getState() {  
        return state;
}

protected final void setState(int newState) { 
        state = newState;
}

protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```
#### 1.2 AQS 对资源的共享方式
AQS定义两种资源共享方式
- 独占：只有一个线程能持有资源，如 `ReentrantLock`。又可分为公平锁和非公平锁： 
    - 公平锁：按照线程在队列中的排队顺序，先到者先拿到锁；
    - 非公平锁：当线程要获取锁时，无视队列顺序直接去抢锁，谁抢到就是谁的。
- 共享：多个线程可同时持有资源。
#### 1.3 模板方法模式
使用者继承 `AbstractQueuedSynchronizer` 并重写指定的方法。
将 AQS 组合在自定义同步组件的实现中，并调用其模板方法，而这些模板方法会调用使用者重写的方法。
```java
//该线程是否正在独占资源。只有用到condition才需要去实现它。
isHeldExclusively() 

//独占方式。尝试获取资源，成功则返回true，失败则返回false。
tryAcquire(int) 

//独占方式。尝试释放资源，成功则返回true，失败则返回false。
tryRelease(int) 

//共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
tryAcquireShared(int) 

//共享方式。尝试释放资源，成功则返回true，失败则返回false。
tryReleaseShared(int) 
```
默认情况下，每个方法都抛出 `UnsupportedOperationException`。 这些方法的实现必须是内部线程安全的，并且通常应该简短而不是阻塞。AQS 类中的其他方法都是 final ，所以无法被其他类使用，只有这几个方法可以被其他类重写。
以 `ReentrantLock` 为例，它的 `state` 初始化为 0，表示没有线程获取锁。A 线程 `lock()` 时，会调用 `tryAcquire()` 独占该锁并将 state+1。此后，其他线程再 tryAcquire() 时就会失败，直到 A 线程 unlock 到 state=0 为止，其它线程才有机会获取该锁。
当然，释放锁之前，A 线程自己是可以重复获取此锁的(state 会累加)，这就是可重入的概念。但要注意，获取多少次就要释放多么次，这样才能保证 state 是能回到零态的。
### 2 AQS 数据结构
AQS 类底层的数据结构是使用 CLH 队列是一个虚拟的双向队列(虚拟的双向队列即不存在队列实例，仅存在结点之间的关联关系)。
AQS 是将每条请求共享资源的线程封装成一个 CLH 锁队列的一个结点来实现锁的分配。其中 `Sync queue`，即同步队列，是双向链表，包括 head结点和 tail 结点，head 结点主要用作后续的调度。
`Condition queue` 并不是必须的，其是一个单向链表，只有当使用 `Condition` 时，才会存在此单向链表。并且可能会有多个 `Condition queue`。
![[AQS 数据结构.png]]
### 3 AQS 源码分析
#### 3.1 类继承关系
`AbstractQueuedSynchronizer` 继承自 `AbstractOwnableSynchronizer` 抽象类，并且实现了 `Serializable` 接口，可以进行序列化。
```java
public abstract class AbstractQueuedSynchronizer  
    extends AbstractOwnableSynchronizer  
    implements java.io.Serializable
```
其中 `AbstractOwnableSynchronizer` 类的源码如下：
```java
public abstract class AbstractOwnableSynchronizer
    implements java.io.Serializable {
    
    private static final long serialVersionUID = 3737899427754241961L;

    protected AbstractOwnableSynchronizer() {}

    private transient Thread exclusiveOwnerThread;

    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }
    protected final Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }
}
```
`AbstractOwnableSynchronizer` 抽象类中提供了设置独占资源线程和获取独占资源线程的能力。
#### 3.2 Node 类
```java
static final class Node {
	// ====== 节点模式 ======
	// 共享模式
	static final Node SHARED = new Node();
	// 独占模式
	static final Node EXCLUSIVE = null;

	// ====== 节点状态 ======
	// 标识当然线程被取消
	static final int CANCELLED =  1;
	// 标识当前节点的后继节点包含的线程需要运行
	static final int SIGNAL    = -1;
	// 标识当前接在在等待 Condition，也就是在 Condition 队列中
	static final int CONDITION = -2;
	// 标识当然场景下后续的 acquireShared 能执行
	static final int PROPAGATE = -3;

	// ====== 属性 ======
	// 节点状态
	volatile int waitStatus;
	// 前驱节点
	volatile Node prev;
	// 后继结点
	volatile Node next;
	// 节点锁对应的线程
	volatile Thread thread;
	// 下一个等待者
	Node nextWaiter;

	// 节点是否在共享模式下等待
	final boolean isShared() {
		return nextWaiter == SHARED;
	}

	// 获取前驱节点，如果节点为空，抛出异常
	final Node predecessor() throws NullPointerException {
		Node p = prev;
		if (p == null)
			throw new NullPointerException();
		else
			return p;
	}
	// 无参构造方法，用于建立初始头或共享标记
	Node() { }
	Node(Thread thread, Node mode) { 
		this.nextWaiter = mode;
		this.thread = thread;
	}
	Node(Thread thread, int waitStatus) {
		this.waitStatus = waitStatus;
		this.thread = thread;
	}
}
```
每个线程被阻塞的线程都会被封装成一个Node结点，放入队列。
每个节点包含了一个 Thread 类型的引用，并且每个节点都存在一个状态，具体状态如下。
- `CANCELLED`，值为 1，表示当前的线程被取消；
- `SIGNAL`，值为 -1，表示当前节点的后继节点包含的线程需要运行，需要进行 unpark 操作；
- `CONDITION`，值为 -2，表示当前节点在等待 condition，也就是在 condition queue 中；
- `PROPAGATE`，值为 -3，表示当前场景下后续的 acquireShared 能够得以执行；
- 值为 0，表示当前节点在 sync queue 中，等待着获取锁。
