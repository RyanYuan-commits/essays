### 1 修改核心线程数
设置核心线程数。这将覆盖构造函数中设置的值。如果新值小于当前值，那么当多余的现有线程下一次空闲时，它们将被终止。如果更大，将在需要的时候启动新的线程来执行队列中的任务。
```java
 public void setCorePoolSize(int corePoolSize) {  
    if (corePoolSize < 0 || maximumPoolSize < corePoolSize)  
        throw new IllegalArgumentException();  
    int delta = corePoolSize - this.corePoolSize;  
    this.corePoolSize = corePoolSize;  
    if (workerCountOf(ctl.get()) > corePoolSize)  
        interruptIdleWorkers();  
    else if (delta > 0) {   
        int k = Math.min(delta, workQueue.size());  
        while (k-- > 0 && addWorker(null, true)) {  
            if (workQueue.isEmpty())  
                break;  
        }  
    }  
}
```
首先对入参进行检测，新的核心线程数不能小于零且不能大于最大线程数。
然后对传入的核心线程数 和 目前的核心线程数做一个比较，分别做不同的处理：
#### 1.1 调小核心线程数
也就是新的核心线程数小于此时线程池的核心线程数的情况：
```java
if (workerCountOf(ctl.get()) > corePoolSize)  
	interruptIdleWorkers();  
	
private void interruptIdleWorkers() {  
    interruptIdleWorkers(false);  
}
```
会尝试中断所有的空闲线程，`interruptIdleWorkers()`，最终会调用到这个方法，传入的参数代表要停止的是几个线程：
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
方法中会尝试中断所有的闲置线程，此时，被阻塞等待任务的线程此时会重新调度，因为此时线程数大于设置的核心线程数，这些线程会因为获取不到任务而被销毁。
#### 1.2 调大核心线程数
此时就需要去增加线程了，那具体要增加多少呢？线程池的设计师采用了这么一种启发式的方式：
```java
else if (delta > 0) {        
	 int k = Math.min(delta, workQueue.size());  
	while (k-- > 0 && addWorker(null, true)) {  
		if (workQueue.isEmpty())  
			break;  
	}  
}  
```
将新增的线程个数与当前任务队列中的任务个数取一个最小值，将增量设置为这个值。
在正常情况下，如果任务队列中有任务，核心线程应该是全部处于 alive 状态的，而如果任务队列中无任务，核心线程数不应该继续增加，这样更有利于资源的优化。
### 2 修改最大线程数
```java
public void setMaximumPoolSize(int maximumPoolSize) {  
    if (maximumPoolSize <= 0 || maximumPoolSize < corePoolSize)  
        throw new IllegalArgumentException();  
    this.maximumPoolSize = maximumPoolSize;  
    if (workerCountOf(ctl.get()) > maximumPoolSize)  
        interruptIdleWorkers();  
}
```
这个方法的执行逻辑就是当最大线程数大于当前的线程数的话，就清除掉未在活动的线程。
对于其他的线程，处理方法也在 `getTask()` 中：
```java
private Runnable getTask() {  
		boolean timedOut = false; // Did the last poll() time out?  
		
		for (;;) {  
			int c = ctl.get();  
			// 。。。。。。
			int wc = workerCountOf(c);  
			
			if ((wc > maximumPoolSize || (timed && timedOut))  
				&& (wc > 1 || workQueue.isEmpty())) {  
				if (compareAndDecrementWorkerCount(c))  
					return null;  
			continue;  
		}  
		// 。。。。。。
	}  
}
```
这个当线程数大于最大线程数的时候，线程池只需要保证线程总数最低限度为 1，或者任务队列为空即可，其余的线程会全部返回 null。