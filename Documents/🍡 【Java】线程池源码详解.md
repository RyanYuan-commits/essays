## 1 任务提交
### 1.1 任务提交源码

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
	    // (3)尝试添加非核心线程，但已经达到最大线程数，所以添加失败
	    // 执行拒绝策略
        reject(command);
}
```
- 当一个任务提交到线程池中，线程池首先会检测当前的线程是否达到核心线程数，如果小于核心线程数，直接创建线程，并将这个任务设置为该线程执行的第一个任务
- 而如果当前线程已经大于核心线程数，会首先将任务放入到阻塞队列。
- 如果阻塞队列也已经无法放入任务了，此时就会在最大线程数的限制下去增加新的线程。
- 上述的方式全部失败了，执行拒绝策略。
#### 1.2 关于 ctl
```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```
线程池中的 `ctl` 是一个 `AtomicInteger` 类型的变量，分别用它的高位和低位来存储线程池的状态和线程数目
- 通过 `ctl` 的高位来表示线程池的当前状态 `RUNNING`（运行）、`SHUTDOWN`（关闭）、`STOP`（停止）、`TIDYING`（整理）、`TERMINATED`（终止）。
- 通过 `ctl` 的低位来记录当前线程池中活跃的工作线程的数量。
#### 1.3 关于拒绝策略
```Java
public interface RejectedExecutionHandler {
    void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
}
```
拒绝策略类都实现了 `RejectedExecutionHandler` 接口，这是一个函数式接口。
`ThreadPoolExecutor` 提供了一些拒绝策略的实现：
- `AbortPolicy`：直接抛出一个异常 RejectedExecutionException。
- `CallerRunsPolicy`：任务由提交任务的线程来继续执行。
- `DiscardOldestPolicy`：丢弃阻塞队列中最旧的未处理任务，然后尝试重新提交被拒绝的任务。
- `DiscardPolicy`：默默丢弃掉任务而不做任何的处理。
### 2 任务执行
#### 2.1 Worker 工作线程类
```java
if (workerCountOf(c) < corePoolSize) {
	// (1) 当前线程数小于核心线程数，需要创建线程
	if (addWorker(command, true)) return;
	......
}
```
这是线程池 execute() 方法的片段，当线程数小于核心线程的时候，将任务通过 addWorker() 分配给新创建的线程执行。这些新创建的线程会被存储到 workers 属性中：
```java
private final HashSet<Worker> workers = new HashSet<>();
```

Worker 是对线程池实际的 Thread 的一个拓展，它是 ThreadPoolExecutor 的一个内部类：
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
其中的 final Thread thread 就是实际的线程对象。
而 Worker 继承并拓展了 Thread 类的功能，添加了 completedTasks 等参数，并且通过继承 AbstractQueuedSynchronizer 实现了锁。 
同时，Worker 还实现了 Runnable 接口，在构造方法中，通过 ThreadFactory 将 this 作为参数来创建线程：
```java
public interface ThreadFactory {  
  Thread newThread(Runnable r);  
}
```
这样最终创建出来的线程 **实际执行** 的就是 Worker 实现 Runnable 接口的 run() 方法：
```java
public void run() {  
    runWorker(this);  
}
```

#### 2.2 工作线程的 "run" 方法
当 `Worker` 中的线程被启动并分配到 CPU 资源后，就会开始执行 `Worker` 的 `run()`，并最终执行到 `runWorker(Worker w)` 方法：
```java
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
- `Worker` 会不断的通过 `getTask()` 方法从任务队列中获取任务 **(1)**
- 并最终通过调用 `Runnable` 接口的 `run()` 方法去执行任务 **(2)**
- 如果 `getTask()` 方法返回 null，就会通过 `processWorkerExit(w, completedAbruptly)` 根据当前的情况去执行线程的销毁。 **(3)**
当线程退出循环的时候，就会执行 `processWorkerExit(w, completedAbruptly)`，而正常退出循环的条件是 `(task = getTask()) != null`，可见这个方法除了获取任务的功能外，还承载了工作线程生命周期管理的功能。

#### 2.3 getTask 任务获取与线程管理
##### 源码解析
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
		// (2) 降低线程总数并且，返回 null（这个线程将会被销毁）
        if ((wc > maximumPoolSize || (timed && timedOut)) && (wc > 1 || workQueue.isEmpty())) {
	        // CAS 成功，线程需要被销毁了
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
在 getTask 方法中，线程会尝试从任务队列中获取任务，如果这个方法返回为 `null` 则表明线程可以被销毁。
方法中有两个重要的变量，`timed` 和 `timeOut`：
- 当用户允许核心线程的超时销毁 或者 当前线程数大于核心线程数的时候，`timed` 为 `true`，表明当前的线程可以因为等待超时而被销毁。
- 当前线程等待时间已经超时的时候，`timeOut` 会被设置为 true，在执行下一次循环的时候，getTask 方法返回 null。

方法中根据当前线程是否超时销毁执行两种获取方法：`workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS)` 和 `workQueue.take()`
- `workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS)`  以 `keepAliveTime` 为参数，在这段时间内如果没有获取到任务的话，会停止等待。
- `workQueue.take()` 是一个持久性的挂起，即使长时间没有获取到任务，线程也会处于一个挂起的状态而不会返回。
当线程返回后，如果获取到了任务，直接返回，而如果是超时返回，会将 `timedOut` 设置为 `true`，在下一次循环的时候，如果 timed 和 timeOut 同时为 true，这个线程就要被销毁了，通过 CAS 减少线程数然后返回 `null`。
##### 小结
调用这个方法的线程会尝试从任务队列中获取任务，如果这个方法返回 null，这个 Worker 就会被销毁；
线程数目小于核心线程数目，这个方法会阻塞线程直到获取到任务，这确保了核心线程不会被销毁；
而如果此时线程数目大于核心线程数目，此时调用这个方法的时候，会阻塞 keepAliveTime 时间然后返回，这个线程就会被销毁，线程池通过这样的方式来控制非核心线程的存活时间的。

`getTask` 方法返回 null 的时候，表示这个线程需要被销毁了，在这些情况下，方法会返回 `null`：
- 当前线程池的状态为 `STOP`。
- 当前线程池的状态是 `SHUTDOWN` 且任务队列为空。
- 当前线程为非核心线程，且在超时时间内未获取到任务。
通过前两种情况，其实可以窥探到线程池的生命周期控制是如何实现的，其实就是设置一个标志（前面提到的 ctl），getTask() 方法中，会根据线程池的状态做出反应。

### 3 线程的销毁、线程异常退出的处理
上面的 getTask() 方法返回 null 之后，就会执行 processWorkerExit 方法，除此之外，当线程抛出异常后，同样会执行这个方法：
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
当循环正常退出的时候，completedAbruptly 的值为 false；而如果因为抛出异常而退出，completedAbruptly 的值会为 true。
#### 3.1 循环正常退出
- 统计当前完成的任务数量；
- 通过 workers.remove(w); 语句来删除当前的 Worker，
- 然后去判断当前线程数目是否能满足最小值，如果满足，直接返回。
#### 3.2 抛出异常退出
如果是由于抛出异常被捕获导致的异常退出，会通过 `addWorker` 去再创建一个新的线程，顶替原来抛出异常的线程。
也就是对于抛出异常的情况，线程池会重新创建一个新的 `Worker`，那为什么不干脆捕获异常然后让这个线程继续运行呢？
这样做是没问题的，但如果不将异常给抛出的话，`Thread` 自带的捕获异常机制就无法触发，这个机制可以让我们在线程抛出异常后做一些特殊的处理，线程工厂在创建线程的时候，可以指定一个 `Handler`。
```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(1, 1, 0L, TimeUnit.DAYS, new LinkedBlockingQueue<Runnable>(),
        r -> {
            Thread thread = new Thread(r);
            thread.setUncaughtExceptionHandler((t, e) -> System.out.println("Uncaught exception: " + e.getMessage()));
            return new Thread(r);
        },new ThreadPoolExecutor.AbortPolicy());
```
线程池将任务执行过程中抛出异常后如何处理开放给了开发者，所以选择了抛出异常然后重新创建的方式。
### 4 线程池的关闭
线程池提供了两种关闭方法，`shutdown` 方法会等待所有任务结束后再关闭线程池，而 `shutdownNow` 则会立刻关闭线程池。
这两种方法会改变线程池的状态标志位，然后通过 `interrupt` 而不是 `stop` 方法来尝试中断所有闲置的 Worker 线程，这样就避免了线程在执行过程中的突然中断引发意料之外的问题；线程池中的线程是通过 `ReentrantLock` 来挂起的，其底层调用了 LockSuport 的挂起方式，通过这种方式挂起的线程，在被中断后不会抛出中断异常，而是将中断标志位改为 true 后重新被调度，
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
在 `interruptIdleWorkers` 方法中，会遍历所有的线程，并尝试获取 `Worker` 的锁，如果获取成功，就会调用 `interrupt` 中断方法。
在 `getTask` 方法中挂起等待任务获取的线程在被调用中断方法后会被唤醒，如果线程处于 `shutdown` 的话，还会再检查一次任务队列是否为空，而如果是 `stop` 状态，不会判断任务队列是否为空，直接销毁线程。
