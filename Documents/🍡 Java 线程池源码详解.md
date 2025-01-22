![[Java 线程池源码详解思维导图.png]]
# 任务提交
![[线程池任务提交流程图.png]]
测试用例：
```java
public class Main {  
    public static void main(String[] args) {  
        ThreadPoolExecutor poolExecutor = new ThreadPoolExecutor(1, 1, 100,  
                TimeUnit.SECONDS, new LinkedBlockingQueue<>(1024), Thread::new, new ThreadPoolExecutor.AbortPolicy());  
        poolExecutor.execute(()->{  
            System.out.println("hello");  
        });  
    }  
}
```

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
- 当一个任务提交到线程池中，线程池首先会检测当前的线程是否达到核心线程数，如果小于核心线程数，直接创建线程，并将这个任务设置为该线程执行的第一个任务。**(1)**
- 而如果当前线程已经大于核心线程数，会首先将任务放入到阻塞队列。**(2)**
- 如果阻塞队列也已经无法放入任务了，此时就会在最大线程数的限制下去增加新的线程。**(3)**
- 上述的方式全部失败了，就会执行我们配置的拒绝策略。**(4)**

> [!tip] （1）关于核心参数 ctl
```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```
线程池中的 `ctl` 是一个 `AtomicInteger` 类型的变量，分别用它的高位和低位来存储线程池的状态和线程数目
- 通过 `ctl` 的高位（通常是高3位）来表示线程池的当前状态 `RUNNING`（运行）、`SHUTDOWN`（关闭）、`STOP`（停止）、`TIDYING`（整理）、`TERMINATED`（终止）。
- 通过 `ctl` 的低位来记录当前线程池中活跃的工作线程的数量。

> [!tip] （2） 关于拒绝策略
```Java
public interface RejectedExecutionHandler {
    void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
}
```
拒绝策略需要实现接口：RejectedExecutionHandler 这是一个函数式接口，规范了 rejectedExecution 执行方法。
在原生的 JDK 中，这个接口有这些实现类，这些类都是 ThreadPoolExecutor 类的 **静态内部类**。
	1）AbortPolicy：直接抛出一个异常 RejectedExecutionException。
	2）CallerRunsPolicy：任务由提交任务的线程来继续执行。
	3）DiscardOldestPolicy：丢弃阻塞队列中最旧的未处理任务，然后尝试重新提交被拒绝的任务。
	4）DiscardPolicy：默默丢弃掉任务而不做任何的处理。

---

任务提交的流程：
	参数检测
	判断当前线程数和核心线程数的关系：如果小于，创建新的线程，然后将任务交给它执行；如果大于，放到阻塞队列
	阻塞队列已满：而线程数未达到最大线程，创建新的线程来执行；否则执行拒绝策略。

---
# 任务执行
## 关于 Worker
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
## 实际执行任务：runWorker(Worker w)
当 `Worker` 中的线程被启动并分配到 CPU 资源后，就会开始执行 `Worker` 的 `run()`，并最终执行到本节要讲的 `runWorker(Worker w)` 方法：
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
            if ((runStateAtLeast(ctl.get(), STOP) || (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) && !wt.isInterrupted())
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
为了保证严谨，我这里贴出整个方法的完整代码，但是本节关注的是实际任务是如何执行的，所以不会将方法完全讲完，在阅读本节的时候只需要关注标注了数字序号的部分即可。
Worker 会不断的通过 getTask() 方法从任务队列中获取任务 **(1)**
并最终通过调用 Runnable 接口的 run() 方法去执行任务 **(2)**
如果 getTask() 方法返回 null，就会通过 processWorkerExit(w, completedAbruptly); 根据当前的情况去执行线程的销毁。 **(3)**
- 并不是任务队列一空就立刻结束循环，然后执行 processWorkerExit() 方法；
- getTask() 方法在获取不到任务的时候并不会立刻返回，而是会阻塞线程一段时间。**(4)**
- 在这里我们需要记住，getTask() 方法返回 null 的时候，代表着线程将要被销毁，getTask() 方法对线程的生命周期控制尤为重要。
抛出异常退出和正常销毁都会在 processWorkerExit(w, completedAbruptly) 方法中去处理。
## 获取任务 getTask() 方法
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
        // (1) 当允许销毁核心线程和当前线程数大于核心线程数的时候，这个值为 true
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
            // (4) 线程获取任务超时，这个线程会在下个循环中于代码 (3) 位置得到 null
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```
这个方法中是一个无限循环，里面有两个重要的局部变量：timed 和 timeOut 这两个变量分别标识当前的 Worker 是否会过期和当前 Worker 是否已经过期；
timed 在两种情况下会为 true，非核心线程的 timed 一定会为 true，而核心线程在用户允许销毁核心线程的时候也为 true；
方法中根据当前线程是否会过期，会执行两种不同的方法：workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) 和 workQueue.take() **(3)**
- 其中 workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS)  以 keepAliveTime 为参数，在这段时间内如果没有获取到任务的话，也会返回。
- 而 workQueue.take() 是一个持久性的挂起，即使长时间没有获取到任务，线程也会处于一个挂起的状态而不会返回。
当线程返回后，如果获取到了任务，直接返回。
反正，则是超时返回的情况，此时，将 timedOut 设置为 true。**(4)**
在下一次循环的时候，如果 timed 和 timeOut 同时为 true，并且此时线程数大于核心线程数，这个 Worker 就要被销毁了；在循环中不断重试 CAS，并且返回 null。
这个方法是否返回 null 对 Worker 的生命周期影响很大，在以下的情况中，会返回 null。
- 当前线程池的状态为 STOP。
- 当前线程池的状态是 SHUTDOWN 且任务队列为空。
- 当前线程为非核心线程，且在超时时间内未获取到任务。
通过前两种情况，其实可以窥探到线程池的各种状态是如何实现的，其实就是设置一个标志（前面提到的 ctl），然后在诸如 getTask() 这样的方法，去不断检查线程池的状态，并做出反应。

---

关于 getTask 方法的小结：该方法实现了从任务队列中获取任务，如果这个方法返回 null，这个 Worker 就会被销毁；
线程数目小于核心线程数目，这个方法会阻塞线程直到获取到任务，这确保了核心线程不会被销毁；
而如果此时线程数目大于核心线程数目，此时调用这个方法的时候，会阻塞 keepAliveTime 时间然后返回，这个线程就会被销毁，线程池通过这样的方式来控制非核心线程的存活时间的。

---
# 线程的销毁、线程异常退出的处理
上面的 getTask() 方法返回 null 之后，就会执行 processWorkerExit(w, completedAbruptly); 语句；当线程抛出异常而退出的时候，同样会执行这个语句：
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
上面提到的两种情况都会执行这个方法，分别看一下这两种情况是如何执行的：
由于 getTask() 返回 null，正常退出的情况，此时的 completedAbruptly 为 false：
	统计当前完成的任务数量
	通过 workers.remove(w); 语句来删除当前的 Worker。
	然后去判断当前线程数目是否能满足最小值，如果满足，直接返回
如果是由于抛出异常被捕获导致的异常退出，并不会执行 if (!completedAbruptly) 代码块中的语句，而是  addWorker(null, false); 去再创建一个新的线程。
也就是对于抛出异常的情况，线程池选择将其销毁，然后重新创建一个新的 Worker，那为什么要这么做呢？
```java
try {
	task.run();
	afterExecute(task, null);
} catch (Throwable ex) {
	afterExecute(task, ex);
	throw ex;
}
```
上面是 Worker 执行任务的方法，异常会被直接抛出；如果直接 catch 住，然后让这个工作线程继续执行任务不是更好吗？
这样做是没问题的，但是如果不将异常给抛出的话，Thread 自带的捕获异常机制就无法触发，这个机制可以让我们在线程抛出异常后做一些特殊的处理。
线程工厂在创建线程的时候，可以指定一个 Handler。
```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(1, 1, 0L, TimeUnit.DAYS, new LinkedBlockingQueue<Runnable>(),
        r -> {
            Thread thread = new Thread(r);
            thread.setUncaughtExceptionHandler((t, e) -> System.out.println("Uncaught exception: " + e.getMessage()));
            return new Thread(r);
        },new ThreadPoolExecutor.AbortPolicy());
```
线程池将任务执行过程中抛出异常后如何处理开放给了开发者，所以选择了抛出异常然后重新创建的方式。
# 线程池的关闭
线程池提供了两种关闭方法：
- ThreadPoolExecutor.shutdown()：等待任务执行完关闭。
- ThreadPoolExecutor.shutdownNow()：立刻关闭线程池。
这这两种方法会使用 Thread.interrupt() 来尝试中断所有的 Worker 线程，而不是通过 Thread.stop()。这样就避免了线程在执行过程中的突然中断引发的问题，同时，开发者可以在任务代码中判断线程的 interrupted 状态来做一些自己的操作。这两个方法都是改变线程池状态的标志位，并且执行中断所有 Worker 的方法。
```java
public void shutdown() {  
    final ReentrantLock mainLock = this.mainLock;  
    mainLock.lock();  
    try {  
        checkShutdownAccess();  
        // 设置线程池状态
        advanceRunState(SHUTDOWN);  
        // 尝试中断所有的 Worker
        interruptIdleWorkers();  
        onShutdown(); // hook for ScheduledThreadPoolExecutor  
    } finally {  
        mainLock.unlock();  
    }  
    tryTerminate();  
}
```
interruptIdleWorkers() 方法：
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
在 interruptIdleWorkers() 方法中，会尝试去获取 Worker 的锁，并且设置它的中断标志位。
在 getTask() 方法中有中断线程的核心部分，通过 return null 的时机来控制是否要将任务执行完
	如果是 shutdown 的话，会在没有任务的时候返回 null
	而如果是 stop 的话，不需要判断任务队列是否为空，会直接返回 null
当返回 null 之后就可以在 runWorker 中安全的执行线程的销毁了。
```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();

        // 如果线程池处于 STOP 的状态 或者 任务队列为空，返回 null
        if (runStateAtLeast(c, SHUTDOWN)
            && (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
            decrementWorkerCount();
	            return null;
        }
        
        // ......
        
        try {
            // 防止抛出 InterruptedException 异常影响正常流程
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null) return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```
当线程执行 interrupt() 方法后，被挂起的线程将会苏醒，在 shutdown 的情况下，如果获取到任务就继续执行，反之直接退出，在 stop 下会直接退出。
	java 提供的阻塞队列挂起线程的时候，使用的是重入锁 ReentrantLock 的 Condition 内部类的 await() 方法，方法内部使用的是 LockSupport 来进行线程的挂起。通过这种方法挂起的线程被中断的时候不会中断异常，而是仅仅设置标志位然后唤醒线程。
	对于自定义的队列也会通过下面的 catch 捕获异常，确保线程会回到 runWorker() 方法中通过 processWorkerExit() 方法被安全的删除。
