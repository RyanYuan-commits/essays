## 1 为什么需要线程池？

和连接池相同的池化思想, 复用思想.

创建和销毁线程的开销很大, 通过重用已有的线程可以 **减少资源消耗**, 同时还能对线程进行统一管理和控制.

## 2 线程池参数

![[线程池参数.png|700]]

### 2.1 任务队列

线程池中使用的任务队列为 `BlockingQueue<Runnable>` 的实现类, 默认提供的有:

- `ArrayBlockingQueue`: 基于数组的固定大小阻塞队列, 通过限制长度可以避免任务的无限量增长.
- `LinkedBlockingQueue`: 基于链表的容量可选阻塞队列, 默认容量为 `Integer.MAX_VALUE`, 适合任务处理速度波动较大时使用.
- `SynchronousQueue`: 不存储元素的同步队列, 队列中有元素的时候, 其他元素的 `put` 方法会一直阻塞, 直到其他线程从队列中通过 `take` 取出元素.
- `PriorityBlockingQueue`: 无界阻塞队列, 按照元素的优先级进行排序, 优先级最高的元素最先被取出, 比较基于 `Comparator` 接口.
- `DelayQueue`: 无界阻塞队列, 插入时指定过期时间, 元素必须等待延迟时间到期后才能被取出.

### 2.2 线程池的拒绝策略

当任务队列已满, 且线程池内的线程数达到最大线程数的时候, 会触发拒绝策略.

```Java
public interface RejectedExecutionHandler {
    void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
}
```

拒绝策略是实现了 `RejectedExecutionHandler` 接口的类, 在 `ThreadPoolExecutor` 中, 提供了一些默认拒绝策略实现, 具体有:

- `AbortPolicy`: 抛出 `RejectedExecutionException` 异常;
- `CallerRunsPolicy`: 任务由提交任务的线程来继续执行;
- `DiscardOldestPolicy`: 丢弃阻塞队列中最旧的任务, 然后尝试重新提交被拒绝的任务;
- `DiscardPolicy`: 默默丢弃掉任务而不做任何的处理.

## 3 Executors 创建的线程池

`Executors` 是 Java 并发包提供的工具类, 提供了一系列的静态工厂方法，用于创建不同类型的 `ExecutorService` 线程池实例。

### 3.1 默认实现

- `FixedThreadPool` 创建一个固定大小的线程池. 其核心线程数和最大线程数是相同的, 采用的阻塞队列为无界队列.
- `SingleThreadExecutor` 创建一个只有一个线程的线程池. 保证所有任务都会按照提交的顺序串行执行, 这个线程池非常适合需要确保任务顺序执行的场景, 线程的管理可以交给线程池来执行, 当线程因为异常而中断的时候, 线程池会创建一个新的线程来替代它.
- `CachedThreadPool` 创建一个可缓存的线程池. 这个线程池的核心线程数为 0, 最大线程数为 `Integer.MAX_VALUE`, 线程的最大存活时间为 60s. 这个线程池的优点是高效, 适合执行大量短期的异步任务. 但由于线程数量是动态变化的, 如果任务提交速率过快, 可能会创建过多的线程, 导致系统资源耗尽. 
- `ScheduledThreadPool` 创建一个固定大小的线程池, 用于支持定时和周期性任务的执行. 它提供了 `schedule()`、`scheduleAtFixedRate()` 和 `scheduleWithFixedDelay()` 等方法, 可以用来在指定延迟后或按照固定的时间间隔来执行任务. 它适用于需要执行后台定时任务的场景. 

### 3.2 为什么不推荐?

`Executors` 提供了便捷的线程池创建功能, 但是它隐藏了线程创建的关键参数, 可能会导致一些意想不到的资源耗尽问题.

比如 `FixedThreadPool`, `SingleThreadExecutor` 还有 `newScheduledThreadPool`, 都使用了无界队列, 当任务提交速度超过任务处理速度的时候, 任务会堆积在队列中, 可能会导致内存溢出.

`CachedThreadPool` 最大线程数为 `Integer.MAX_VALUE`, 当如果提交了大量的任务且任务执行时间都比较短, 系统可能会创建无数个线程来执行这些任务, 导致内存溢出.

## 4 线程池生命周期

![[Java-线程池生命周期.png]]

1. `RUNNING` → `SHUTDOWN`: 通过 `shutdown` 方法触发. 线程池停止接收新任务, 但已提交的任务仍会执行完毕.
2. `SHUTDOWN` → `STOP`: 通过 `shutdownNow` 方法触发. 线程池尝试停止所有正在执行的任务，并清空任务队列中的待执行任务.
3. `SHUTDOWN` → `TIDYING`: 当线程池停止接收新任务, 并且所有任务执行完成, 线程池中没有线程在运行时, 进入 `TIDYING` 状态.
4. `TIDYING` → `TERMINATED`: 当线程池中的所有线程结束并且所有资源被清理完, 线程池进入 `TERMINATED` 状态.

