ReentrantLock 是一个可重入的独占锁，同时只要一个线程获取到锁，其他线程获取锁的时候，会被阻塞并且放入锁的 AQS 阻塞队列；ReentrantLock 中实现了公平锁和非公平锁，使用默认构造方法得到的是非公平锁，类中提供了一个传入布尔值的构造方法，`ReentrantLock(boolean fair)` 使用该构造方法传入 `true` 得到的就是公平锁。
### 1 基本介绍
```java
public ReentrantLock() {
	sync = new NonfairSync();
}
public ReentrantLock(boolean fair) {
	sync = fair ? new FairSync() : new NonfairSync();
}
```
`ReentrantLock` 中提供了两种构造方法，来构造公平锁或非公平，具体实现这些方法的类是它的内部类 `Sync`，`Sync` 直接继承了 AQS ，Sync 的两个子类 `FairSync` 和 `NonfairLock` 分别实现了公平锁和非公平锁。
在这里 AQS 队列中的 state 代表锁的可重入次数，默认的情况下，state 的值为 0，表示锁没有被任何线程持有，当第一个线程获取锁的时候会尝试使用 CAS 操作将 state 的值设置为 1，如果这个操作成功了就将锁的持有者设置为这个线程。在第二次获取锁的时候这个值会被设置为 1，当线程释放锁的时候会使 state 减一，直到减为 0 的时候，线程将锁释放。
AQS 中的 state 是用 `int` 声明的，重入的最大次数就是 `int` 能表示的最大值，如果超过这个数字，就会因为越界使其变为负数，此时会抛出异常；
```java
/**
 * The synchronization state.
 */
private volatile int state;
```
重入达到上限的时候会报错 `java.lang.ErrorCreate:Maximum lock count exceeded`。
### 2 获取锁
#### 2.1 void lock() 方法
```java
public void lock() {
	sync.lock();
}
```
这个方法实际上是委托给了 sync 执行的，sync 会在初始化的时候根据传入的值设置为公平锁或非公平锁，非公平锁的 lock 方法是这样的：
```java
/**
 * Performs lock.  Try immediate barge, backing up to normal
 * acquire on failure.
 */
final void lock() {
	if (compareAndSetState(0, 1)) setExclusiveOwnerThread(Thread.currentThread());
	else acquire(1);
}
```
使用 CAS 在 state 预期值为 0 的情况下，将其设置为 1，如果成功了就将当前线程设置为锁的持有者；
反之则会调用 AQS 抽象类中的 acquire(1) 方法来将尝试将线程挂起到阻塞队列中。
##### 非公平锁 tryAcquire 方法
```java
public final void acquire(int arg) {
if (!tryAcquire(arg) &&
	acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
	selfInterrupt();
}
```
这个方法中首先调用了 `tryAcquire(int acquires)` 方法去尝试获取锁；
如果获取失败了，就将线程放到阻塞队列中并挂起，AQS 将 `tryAcquire` 方法开放给子类实现，子类可以自定义这个方法的逻辑来实现不同的锁；
非公平锁中，`tryAcquire` 最终会调用 `nonfairTryAcquire` 方法：
```java
final boolean nonfairTryAcquire(int acquires) {
	final Thread current = Thread.currentThread();
	int c = getState();
	// 锁没有被占有
	if (c == 0) {
	// 采用 CAS 操作修改 state
		if (compareAndSetState(0, acquires)) {
			setExclusiveOwnerThread(current);
			return true;
		}
	}
	// 锁被当前线程持有
	else if (current == getExclusiveOwnerThread()) {
		int nextc = c + acquires; // 增加 state
		 // 越界的情况
		if (nextc < 0) throw new Error("Maximum lock count exceeded");
		setState(nextc);
		return true;
	}
	// 以上两种情况都不符合，返回 false
	return false;
}
```
线程尝试去获取锁，使用 CAS 操作尝试将 state 值从 0 修改为 1
- 如果修改成功将锁的持有者设置为该线程
- 然后判断是否是当前线程持有锁，如果是，重入次数加一。
##### 公平锁 tryAcquire 方法
再来补充一下公平锁的 `tryAcquire()` 方法的实现：
```java
protected final boolean tryAcquire(int acquires) {
	final Thread current = Thread.currentThread();
	int c = getState();
	if (c == 0) {
		// 判断队列中是否有前置的线程，如果有则不会去尝试获取锁
		if (!hasQueuedPredecessors() &&
			compareAndSetState(0, acquires)) {
			setExclusiveOwnerThread(current);
			return true;
		}
	}
	else if (current == getExclusiveOwnerThread()) {
		int nextc = c + acquires;
		if (nextc < 0)
			throw new Error("Maximum lock count exceeded");
		setState(nextc);
		return true;
	}
	return false;
}
}
```
#### 2.2 void lockInterruptibly() 方法
这个方法和 lock 方法类似，它的不同之处在于，使用这个方法获取锁的线程对于 interrupt() 方法会做出立即反应，即抛出 InterruptedException 异常然后返回，而 lock 方法则是在阻塞结束后再去判断线程在挂起的过程中是否出现阻塞。
```java
public void lockInterruptibly() throws InterruptedException {
	sync.acquireInterruptibly(1);
}
```
实际调用了抽象类中的 `acquireInterruptibly` 方法：
```java
public final void acquireInterruptibly(int arg)
		throws InterruptedException {
	// 如果线程被中断，直接抛出异常
	if (Thread.interrupted()) 
		throw new InterruptedException();
	if (!tryAcquire(arg))
	// 调用 AQS 的可终端方法
		doAcquireInterruptibly(arg);
}
```
因为 AQS 底层是使用 LockSupport 来挂起线程的，这样挂起的线程在被中断后不会抛出中断异常，而是会将中断标志设置为 true，留给线程自己去判断然后处理。
#### 2.3 tryLock 方法
```java
public boolean tryLock() {
	return sync.nonfairTryAcquire(1);
}
final boolean nonfairTryAcquire(int acquires) {
	final Thread current = Thread.currentThread();
	int c = getState();
	if (c == 0) {
		if (compareAndSetState(0, acquires)) {
			setExclusiveOwnerThread(current);
			return true;
		}
	}
	else if (current == getExclusiveOwnerThread()) {
		int nextc = c + acquires;
		if (nextc < 0) // overflow
			throw new Error("Maximum lock count exceeded");
		setState(nextc);
		return true;
	}
	return false;
}
```
尝试获取锁，如果锁没有被其他线程持有，当前线程获取到锁并且返回 true，反之则返回 false；
当获取失败了时候，这个方法并不会将线程放到阻塞队列。
#### 2.4 可超时 tryLock
尝试获取锁，与上面方法不同的是，方法设置了超时时间，如果这个时间内还没有获取到锁，则会直接返回 false。
```java
lock.tryLock(1000, TimeUnit.SECONDS);
```
```java
    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
            // 调用 AQS 中的 tryAcquireNanos 方法
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
```
### 3 释放锁
```java
    public void unlock() {
        sync.release(1);
    }
    
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
调用了 AQS 队列中 release 方法，方法中调用了 tryRelease 方法来尝试释放锁，同样，这个方法没有在抽象类中实现，而是在子类中去实现，这个方法最终会返回锁是否空闲，如果空闲的话执行 unparkSuccessor 方法去解除下一个可被唤醒线程的挂起状态，让它们去争夺锁。
```java
protected final boolean tryRelease(int releases) {
	int c = getState() - releases;
	// 如果锁不被当前线程持有，则抛出异常
	if (Thread.currentThread() != getExclusiveOwnerThread())
		throw new IllegalMonitorStateException();
	boolean free = false;
	// 如果 state 变为了 0，释放锁
	if (c == 0) {
		free = true;
		// 清空锁的持有者
		setExclusiveOwnerThread(null);
	}
	setState(c);
	return free; // 锁是否被线程持有
}
```

```java
/**
 * 如果存在的话，唤醒 Successer 线程
 */
private void unparkSuccessor(Node node) {
	/*
	 * 在进行唤醒操作之前，尝试清零等待状态以便为唤醒操作做准备，
	 * 即使这种操作可能失败或者状态在操作过程中被修改，也是可以接受的。
	 */
	int ws = node.waitStatus;
	if (ws < 0)
		compareAndSetWaitStatus(node, ws, 0);

	/*
	 * 要接触挂起状态的线程一般是下一个线程，但如果下一个线程被取消
	 * 或者为空，则从尾部开始遍历查找可以解除的线程
	 */
	Node s = node.next;
	if (s == null || s.waitStatus > 0) {
		s = null;
		for (Node t = tail; t != null && t != node; t = t.prev)
			if (t.waitStatus <= 0)
				s = t;
	}
	if (s != null)
		LockSupport.unpark(s.thread);
}
```
### 4 使用 ReentrantLock 的好处
1. **可定时的锁等待**：**`ReentrantLock`** 提供了 **`tryLock(long timeout, TimeUnit unit)`** 方法，可以在指定的时间内等待获取锁。这在需要避免线程长时间等待的情况下很有用。
2. **可中断的锁等待**：**`ReentrantLock`** 提供了 **`lockInterruptibly()`** 方法，允许在等待锁的过程中响应中断，这样可以避免线程长时间阻塞。
3. **公平锁和非公平锁**：**`ReentrantLock`** 提供了公平锁和非公平锁两种模式，可以通过构造函数来选择。公平锁会按照线程请求锁的顺序来获取锁，而非公平锁则不保证顺序。而 **`synchronized`** 关键字锁默认是非公平的。
4. **可重入性**：**`ReentrantLock`** 是可重入锁，同一个线程可以多次获取同一个锁而不会死锁。而 **`synchronized`** 关键字锁也是可重入的，但是必须在同一个方法或者代码块中。
5. **条件变量**：**`ReentrantLock`** 提供了 **`Condition`** 接口（AQS 提供的），可以设置多个 Condition 对象来更好的控制线程的状态。
### 5 ReentrantLock vs synchronized
对比 `synchronized`，`ReentrantLock` 有这些优势：
- 可以支持公平锁；
- 可以让线程在等待锁的时候响应中断；
- 线程等待的时间是可控的，提供了尝试获取锁（失败不挂起）和指定等待时间的方法。