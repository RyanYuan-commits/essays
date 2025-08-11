---
type: Java 基础
finished: "false"
---

## 1 任务提交

### 1.1 execute 方法

![[线程池任务提交流程图.png]]

```java
public void execute(Runnable command) {
	// 空任务校验
    if (command == null) throw new NullPointerException();
    int c = ctl.get(); // 描述线程池状态的重要变量
    if (workerCountOf(c) < corePoolSize) {
	    // (1) 当前线程数小于核心线程数，需要创建线程
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // (2) 放入阻塞队列 workQueue.offer(command)
    if (isRunning(c) && workQueue.offer(command)) {
	    // corePoolSize 满了 且 任务成功加入任务队列
        int recheck = ctl.get();
        if (!isRunning(recheck) && remove(command))
	        // 如果线程池不处于运行状态，拒绝策略
            reject(command);
        else if (workerCountOf(recheck) == 0)
	        // 如果工作线程为 0，增加线程
            addWorker(null, false);
    } else if (!addWorker(command, false))
	    // (3)尝试添加非核心线程, 但已经达到最大线程数, 所以添加失败
	    // 执行拒绝策略
        reject(command);
}
```

- 当一个任务提交到线程池中, 线程池首先检测当前的线程是否达到核心线程数, 如果小于核心线程数, 直接创建线程, 并将该任务交给新线程执行.
- 而如果当前线程已经大于核心线程数, 将任务放入到阻塞队列.
- 如果阻塞队列已满, 则会尝试添加新的线程, 直到达到最大线程数.
- 如果线程已经达到最大线程数, 且任务队列已满, 则触发拒绝策略.
### 1.2 关于 ctl

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```

`ctl` 是 `ThreadPoolExecutor` 类的成员变量, 用于存储当前线程池实例的状态和线程数量, 其高位存储线程池状态, 低位存储活跃线程数.

## 2 任务执行

### 2.1 Worker 工作线程类

```java
private final HashSet<Worker> workers = new HashSet<>();
```

线程池中的工作线程是以 `Worker` 的形式存在的, 是对实际的 `Thread` 实例的包装, 它们被存储在 `ThreadPoolExecutor` 的 `workers` 属性中.

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
	@SuppressWarnings("serial")
	final Thread thread;  
	
	@SuppressWarnings("serial")
	Runnable firstTask; 

	volatile long completedTasks;  

	 Worker(Runnable firstTask) {  
	    setState(-1); // inhibit interrupts until runWorker  
	    this.firstTask = firstTask;  
	    this.thread = getThreadFactory().newThread(this);  
	}
	// ......
}
```

`thread` 变量中存储的就是实际线程对象, `Worker` 拓展了 `Thread` 的能力;

`Worker` 添加了一些与线程池有关的参数, 如 `completedTasks` 来记录完成的任务数量, `firstTask` 来存储首个任务.

同时, `Worker` 通过继承 `AbstractQueuedSynchronizer` 实现了简单的同步锁;

`Worker` 还实现了 `Runnable` 接口, `thread` 就是以 `Worker` 的 `run` 方法为蓝图构造的.

### 2.2 Worker 的 run 方法

```java
public void run() {  
    runWorker(this);  
}

final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock();
    // 标识当前方法是否为正常退出
    boolean completedAbruptly = true;
    try {
	    // (1) 不断通过 getTask() 获取任务
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // 如果线程池 STOP，阻塞线程；否则唤醒线程
            if ((runStateAtLeast(ctl.get(), STOP) || 
	            (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) && !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                try {
	                // (2) 执行任务
                    task.run();
                    afterExecute(task, null);
                } catch (Throwable ex) {
                    afterExecute(task, ex);
                    throw ex;
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
	    // (3) 处理线程的销毁 ｜ 处理方法异常退出
        processWorkerExit(w, completedAbruptly);
    }
}
```

最终会调用到 `runWorker` 方法, 这就是实际执行任务的位置.

- `Worker` 会不断的通过 `getTask()` 方法从任务队列中获取任务. **(1)**
- 然后会调用任务的 `run()` 方法. **(2)**
- 当退出循环后, 则通过 `processWorkerExit` 根据当前的情况去执行线程的销毁.  **(3)**

当 `getTask` 方法返回 `null` 的时候, 线程正常退出循环并执行 `processWorkerExit` 销毁方法, 可以看出 `getTask` 方法对 `Worker` 生命周期的控制格外重要.

### 2.3 getTask 方法

```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?
    for (;;) {
        int c = ctl.get();
        // 只有在 SHUTDOWN 及以下的状态才检查队列是否为空
        if (runStateAtLeast(c, SHUTDOWN) && (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }
        int wc = workerCountOf(c);
        // (1) 当允许销毁核心线程 和 当前线程数大于核心线程数的时候，这个值为 true
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
		// (2) 降低线程总数并返回 null
        if ((wc > maximumPoolSize || (timed && timedOut)) && (wc > 1 || workQueue.isEmpty())) {
		    // 降低线程数
            if (compareAndDecrementWorkerCount(c)) return null;
            continue;
        }
        try {
            // (3) 如果当前线程为非核心线程，或者核心线程允许销毁，此时最多等待 keepAliveTime，否则永远不销毁
            // 两种阻塞方式分别为：根据 keepAliveTime 阻塞和无限时间的阻塞
            Runnable r = timed ? workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) : workQueue.take();
            if (r != null) return r;
            // (4) 线程获取任务超时
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

`getTask` 方法不只负责从任务队列中获取任务, 还需要对线程池状态的改变做出响应, 判定逻辑比较复杂, 需要分类理解.

#### 基本获取任务流程

```java
boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
```

如果当前线程数大于核心线程数 `corePoolSize`, 将 `timed` 标志变量设为 `true`, 表示当前线程从队列获取数据时会因为超时而退出.

同时, 如果设置允许核心线程超时, `timed` 也会被设置为 `true`.

```java
Runnable r = timed ? workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) : workQueue.take();
```

如果 `timed` 为 `true`, 当获取任务的时候, 会传入 `keepAliveTime` 作为超时时间;反之则调用 `take` 方法, 会一直等待到获取成功或被中断.

```java
if (r != null) return r;
```

如果获取任务成功, 返回当前任务.

```java
if ((wc > maximumPoolSize || (timed && timedOut)) && 
	(wc > 1 || workQueue.isEmpty())) {
	// 降低线程数
	if (compareAndDecrementWorkerCount(c)) return null;
	continue;
}
```

如果获取任务失败, `timed` 标志位会被置为 `true`, 在下次循环的这个 `if` 块中返回 `null`

#### 响应线程池状态变化

```java
if (runStateAtLeast(c, SHUTDOWN) && 
	(runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
	decrementWorkerCount();
	return null;
}
```

具体指的是这一段代码, 当触发后会直接降低线程数并返回 `null`, 触发条件有:

1. 当前线程池状态为 `SHUTDOWN`, 且任务队列为空;
2. 当前线程池状态为 `STOP`

当线程池状态调整为 `SHUTDOWN` 或者 `STOP` 的时候, 会唤醒所有线程, 它们会在下次循环执行判断, 以此来响应线程池状态的变化.

## 3 processWorkerExit 方法

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {  
	// (1) 抛出异常导致的退出，减少线程数量
    if (completedAbruptly) decrementWorkerCount();  
    final ReentrantLock mainLock = this.mainLock;  
    mainLock.lock();  
    try { 
		// (2) 统计完成任务数，并删除 Worker
        completedTaskCount += w.completedTasks;  
        workers.remove(w);  
    } finally {  
        mainLock.unlock();  
    }  
    tryTerminate();  
    int c = ctl.get();  
    if (runStateLessThan(c, STOP)) {  
        if (!completedAbruptly) {  
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;  
            if (min == 0 && ! workQueue.isEmpty())  min = 1;  
            if (workerCountOf(c) >= min)  return; //（3）
        }
		// (4) 添加一个新的 Worker
        addWorker(null, false);  
    }  
}
```

当线程循环结束后, 会执行 `processWorkerExit` 方法来安全销毁, 正常获取任务失败或者抛出异常都会退出循环, 但这两种情况执行的逻辑是不同的.

### 3.1 循环正常退出

此时变量 `completedAbruptly` 标志为 `false`;

```java
completedTaskCount += w.completedTasks;  
workers.remove(w);  
```

统计当前完成的任务数量, 并移除该 `Worker`, 

```java
if (runStateLessThan(c, STOP)) {
	if (!completedAbruptly) {
		int min = allowCoreThreadTimeOut ? 0 : corePoolSize;  
		if (min == 0 && ! workQueue.isEmpty())  min = 1;  
		if (workerCountOf(c) >= min)  return; //（3）
	}
	// (4) 添加一个新的 Worker
	addWorker(null, false);  
}  
```

如果线程池没有到 `STOP` 状态, 且线程数未达到核心线程数, 则会再添加一个新的 `Worker` 来快速的执行完剩余任务.

### 3.2 抛出异常退出

此时 `completedAbruptly` 标志为 `true`.

```java
if (completedAbruptly) decrementWorkerCount();  

if (runStateLessThan(c, STOP)) {  
	if (!completedAbruptly) {  
		// ......
	}
	// (4) 添加一个新的 Worker
	addWorker(null, false);  
}  
```

先降低线程数目, 如果线程池没有达到 `STOP` 状态, 直接新增一个线程.

这里之所以没有选择直接捕获异常让线程继续运行, 是为了支持 `Thread` 类的异常捕获机制;

```java
/**
 * Dispatch an uncaught exception to the handler. This method is
 * intended to be called only by the JVM.
 */
private void dispatchUncaughtException(Throwable e) {
	getUncaughtExceptionHandler().uncaughtException(this, e);
}
```

具体来说, 当创建线程后, 可以通过 `setUncaughtExceptionHandler` 方法, 来为线程指定一个自定义的异常处理器, 这个处理方法会被 `JVM` 调用.

如果线程池直接选择捕获异常, 这个处理器就会失效.

## 4 shutdown 方法

```java
public void shutdown() {  
    final ReentrantLock mainLock = this.mainLock;  
    mainLock.lock();  
    try {  
        checkShutdownAccess();  
        // 设置线程池状态
        advanceRunState(SHUTDOWN);  
        // 尝试中断所有的闲置 Worker
        interruptIdleWorkers();  
        onShutdown(); // hook for ScheduledThreadPoolExecutor  
    } finally {  
        mainLock.unlock();
    }
    tryTerminate();
}
```

`shutdown` 方法会等待线程池中的任务执行完再关闭, 具体的逻辑在 `getTask` 方法中已展示.

```java
private void interruptIdleWorkers(boolean onlyOne) {  
    final ReentrantLock mainLock = this.mainLock;  
    mainLock.lock();  
    try {  
        for (Worker w : workers) {  
            Thread t = w.thread;  
            if (!t.isInterrupted() && w.tryLock()) {  
                try {  
                    t.interrupt();  
                } catch (SecurityException ignore) {  
                } finally {  
                    w.unlock();  
                }  
            }  
            if (onlyOne)  
                break;  
        }  
    } finally {  
        mainLock.unlock();  
    }  
}
```

在 `interruptIdleWorkers` 方法中, 会遍历所有的线程, 并尝试获取 `Worker` 锁, 如果获取成功, 就会调用 `interrupt` 中断该线程, 这些线程会因为中断而被唤醒, 继续执行 `getTask` 方法.

另外, 任务队列底层通过 `LockSupport#park` 方法使线程进入 `WAITING` 状态, 这与 `Object#wait` 或 `Thread#join` 是不同的;

通过 `park` 方法进入 `WAITING` 状态的线程收到中断信号后不会抛出中断异常, 而是将中断标志位设为 `true` 然后返回运行状态, 所以在源码中很多地方没有处理中断异常的方法, 而是不断检查中断标志. 