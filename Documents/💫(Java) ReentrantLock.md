---
type: Concurrent Programming
sub-type: Java
finished: "true"
---
## 1 基本介绍

```java
public ReentrantLock() {
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

`ReentrantLock` 提供了公平锁和非公平锁两种实现, 默认提供非公平锁的实现;

可重入锁是基于 [[🥒(Java) AbstractQueueSynchronizer|AQS]] 实现的, `state` 表示锁被重入的次数, 线程每次成功调用 `lock()` 方法获取到锁都会使 `state` 加一;

AQS 中的 `state` 是 `int` 类型, 重入的最大次数就是 `Integer.MAX_VALUE`, 越界后再操作会抛出异常:

```java
// java.util.concurrent.locks.AbstractQueuedSynchronizer#state
private volatile int state;
```

## 2 获取锁

### 2.1 lock

```java
// java.util.concurrent.locks.ReentrantLock.Sync#lock
@ReservedStackAccess  
final void lock() {  
    if (!initialTryLock())  
        acquire(1);  
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer#acquire(int)
public final void acquire(int arg) {  
    if (!tryAcquire(arg))  
        acquire(null, arg, false, false, false, 0L);  
}
```

获取锁的 `lock` 方法委托给 [[🥒(Java) AbstractQueueSynchronizer|AQS]] 的 `acquire` 执行, 子类只需要实现 `tryAcquire` 方法来定义获取锁的逻辑即可:

```java
// java.util.concurrent.locks.ReentrantLock.NonfairSync#tryAcquire
protected final boolean tryAcquire(int acquires) {  
    if (getState() == 0 && compareAndSetState(0, acquires)) {  
        setExclusiveOwnerThread(Thread.currentThread());  
        return true;  
    }  
    return false;  
}

// java.util.concurrent.locks.ReentrantLock.FairSync#tryAcquire
protected final boolean tryAcquire(int acquires) {  
    if (getState() == 0 && !hasQueuedPredecessors() &&  
        compareAndSetState(0, acquires)) {  
        setExclusiveOwnerThread(Thread.currentThread());  
        return true;  
    }  
    return false;  
}
```

公平锁的 `tryAcquire` 实现会额外检查一下队列中有没有其他的等待者, 以确保公平性.

### 2.2 lockInterruptibly

```java
// java.util.concurrent.locks.ReentrantLock#lockInterruptibly
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer#acquireInterruptibly
public final void acquireInterruptibly(int arg)  
    throws InterruptedException {  
    if (Thread.interrupted() ||  
        (!tryAcquire(arg) && acquire(null, arg, false, true, false, 0L) < 0))  
        throw new InterruptedException();  
}
```

与 `lock` 不同, 使用 `lockInterruptibly` 上锁的线程, 在接收到中断信号后会抛出中断异常.

[[🥒(Java) AbstractQueueSynchronizer|AQS]] 底层使用 `LockSupport` 来挂起线程, 这样挂起的线程在被中断后不会抛出中断异常, 仅会将中断标志设置为 `true`.

### 2.3 tryLock

```java
// java.util.concurrent.locks.ReentrantLock#tryLock()
public boolean tryLock() {  
    return sync.tryLock();  
}

// java.util.concurrent.locks.ReentrantLock.Sync#tryLock
@ReservedStackAccess  
final boolean tryLock() {  
    Thread current = Thread.currentThread();  
    int c = getState();  
    if (c == 0) {
        if (compareAndSetState(0, 1)) {  
            setExclusiveOwnerThread(current);  
            return true;  
        }
    } else if (getExclusiveOwnerThread() == current) {
        if (++c < 0) // overflow
            throw new Error("Maximum lock count exceeded");  
        setState(c);  
        return true;  
    }  
    return false;  
}

// java.util.concurrent.locks.ReentrantLock#tryLock(long, java.util.concurrent.TimeUnit)
public boolean tryLock(long timeout, TimeUnit unit)  
        throws InterruptedException {  
    return sync.tryLockNanos(unit.toNanos(timeout));  
}
```

尝试获取锁, 会直接返回获取成功和失败, 获取失败后线程不会阻塞等待, 重载方法支持指定等待时间.

## 3 释放锁

```java
// java.util.concurrent.locks.ReentrantLock.Sync#tryRelease
@ReservedStackAccess  
protected final boolean tryRelease(int releases) {  
    int c = getState() - releases;  
    if (getExclusiveOwnerThread() != Thread.currentThread())  
        throw new IllegalMonitorStateException();  
    boolean free = (c == 0);  
    if (free)  
        setExclusiveOwnerThread(null);  
    setState(c);  
    return free;  
}
```

调用了 [[🥒(Java) AbstractQueueSynchronizer|AQS]] 的 `release` 方法, 子类只需要实现 `tryRelease` 方法即可.

## 4 Condition

### 4.1 简单介绍

`Condition` 是 `ReentrantLock` 的配套组件, 在应用上层实现了 `Object` 的 `wait`, `notify` 和 `notifyAll` 方法, 用于线程间的协作.

```java
// java.util.concurrent.locks.ReentrantLock#newCondition
public Condition newCondition() {
    return sync.newCondition();
}

// java.util.concurrent.locks.AbstractQueuedSynchronizer#newCondition
final Condition newCondition() {
    return new ConditionObject();
}
```

`ReentrantLock` 的 `newCondition` 方法会返回一个 `ConditionObject` 实例, 这是 [[🥒(Java) AbstractQueueSynchronizer|AQS]] 的一个内部类:

### 4.2 核心方法

| 方法名称                                  | 签名                                                                    | 功能描述                                                      |
| ------------------------------------- | --------------------------------------------------------------------- | --------------------------------------------------------- |
| **`await()`**                         | `void await() throws InterruptedException`                            | 使当前线程释放锁, 进入此 `Condition` 的等待队列, 并阻塞直到被唤醒或中断.             |
| **`signal()`**                        | `void signal()`                                                       | 唤醒在此 `Condition` 上等待队列中的 **一个** 线程, 被唤醒的线程将重新竞争锁.         |
| **`signalAll()`**                     | `void signalAll()`                                                    | 唤醒在此 `Condition` 上等待队列中的 **所有** 线程.                       |
| **`awaitUninterruptibly()`**          | `void awaitUninterruptibly()`                                         | 与 `await()` 类似，但忽略中断。即使线程被中断，它也会保持等待状态.                   |
| **`await(long time, TimeUnit unit)`** | `boolean await(long time, TimeUnit unit) throws InterruptedException` | 带超时等待, 线程最多等待指定时间. 如果等待时间内被唤醒, 返回 `true`; 如果超时返回 `false`. |

## 5 ReentrantLock vs synchronized

对比 `synchronized`, `ReentrantLock` 有这些优势: 

- 可以支持公平锁; 
- 可以让线程在等待锁的时候响应中断; 
- 线程等待的时间是可控的, 提供了尝试获取锁（失败不挂起）和指定等待时间的方法.

