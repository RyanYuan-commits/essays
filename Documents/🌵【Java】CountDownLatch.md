`CountDownLatch` 是 Java 并发包 `java.util.concurrent` 中的一个同步工具类，用于让一个或多个线程等待其他线程完成操作。
### 1 基本概念
`CountDownLatch` 内部维护一个计数器，该计数器的初始值是在创建 `CountDownLatch` 对象时指定的，代表需要等待完成的操作数量。当一个线程完成了自己的任务后，会调用 `countDown()` 方法将计数器减 1。当计数器的值变为 0 时，所有调用 `await()` 方法而处于等待状态的线程将被唤醒，继续执行后续操作。
### 2 使用方法
#### 2.1 构造方法
`CountDownLatch` 只有一个构造方法：
```java
public CountDownLatch(int count)
```
其中，`count` 表示计数器的初始值，即需要等待完成的操作数量，该值必须为正整数。
#### 2.2 主要方法
- **`void await()`**：使当前线程等待，直到计数器的值变为 0。如果计数器已经为 0，则该方法会立即返回。如果在等待过程中当前线程被中断，会抛出 `InterruptedException` 异常。
- **`boolean await(long timeout, TimeUnit unit)`**：使当前线程等待，直到计数器的值变为 0 或者等待时间超过指定的时长。如果在等待时间内计数器变为 0，则返回 `true`；否则返回 `false`。同样，如果在等待过程中当前线程被中断，会抛出 `InterruptedException` 异常。
- **`void countDown()`**：将计数器的值减 1。如果减 1 后计数器的值变为 0，则会唤醒所有因调用 `await()` 方法而处于等待状态的线程。
#### 2.3 示例代码
```java
import java.util.concurrent.CountDownLatch;

public class CountDownLatchExample {
    public static void main(String[] args) throws InterruptedException {
        // 创建一个 CountDownLatch 实例，计数器初始值为 3
        CountDownLatch latch = new CountDownLatch(3);

        // 创建并启动 3 个线程
        for (int i = 0; i < 3; i++) {
            new Thread(new Worker(latch)).start();
        }

        System.out.println("主线程等待所有工作线程完成任务...");
        // 主线程调用 await() 方法等待计数器变为 0
        latch.await();
        System.out.println("所有工作线程已完成任务，主线程继续执行");
    }

    static class Worker implements Runnable {
        private final CountDownLatch latch;

        public Worker(CountDownLatch latch) {
            this.latch = latch;
        }

        @Override
        public void run() {
            try {
                System.out.println(Thread.currentThread().getName() + " 开始执行任务");
                // 模拟任务执行
                Thread.sleep((long) (Math.random() * 1000));
                System.out.println(Thread.currentThread().getName() + " 完成任务");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                // 任务完成后，调用 countDown() 方法将计数器减 1
                latch.countDown();
            }
        }
    }
}
```
- 在 `main` 方法中，创建了一个 `CountDownLatch` 实例，计数器初始值为 3，表示需要等待 3 个线程完成任务。
- 启动 3 个线程，每个线程执行 `Worker` 类的 `run` 方法。
- 在 `run` 方法中，线程模拟执行任务，任务完成后调用 `latch.countDown()` 方法将计数器减 1。
- 主线程调用 `latch.await()` 方法等待计数器变为 0，当所有 3 个线程都调用了 `countDown()` 方法后，计数器变为 0，主线程被唤醒，继续执行后续操作。
### 3 实现原理
`CountDownLatch` 是基于 `AQS`（AbstractQueuedSynchronizer，抽象队列同步器）实现的。`AQS` 是 Java 并发包中许多同步工具的基础，它内部维护了一个状态变量 `state` 和一个等待队列。
在 `CountDownLatch` 中，`state` 变量表示计数器的值。当调用 `countDown()` 方法时，会尝试将 `state` 的值减 1；当调用 `await()` 方法时，如果 `state` 的值不为 0，则当前线程会被放入等待队列中阻塞，直到 `state` 的值变为 0 时，会唤醒等待队列中的所有线程。
### 4 应用场景
- **并行任务同步**：在进行大规模数据处理时，可以将数据分成多个部分，每个线程负责处理一部分数据。主线程使用 `CountDownLatch` 等待所有子线程完成数据处理任务后，再进行后续的汇总操作。
- **资源初始化**：在系统启动时，可能需要初始化多个资源，每个资源的初始化可以由一个线程来完成。主线程使用 `CountDownLatch` 等待所有资源初始化完成后，再启动系统的其他部分。
### 5 与其他并发工具的对比
- **与 `CyclicBarrier` 的对比**：
  - `CountDownLatch` 是一次性的，计数器的值只能减不能增，一旦计数器变为 0，就不能再使用；而 `CyclicBarrier` 是可重用的，当所有线程都通过屏障后，它可以被再次使用。
  - `CountDownLatch` 主要用于一个或多个线程等待其他线程完成任务；而 `CyclicBarrier` 主要用于多个线程之间互相等待，直到所有线程都到达某个公共屏障点。
- **与 `Semaphore` 的对比**：
  - `CountDownLatch` 主要用于线程间的等待和同步；而 `Semaphore` 主要用于控制对有限资源的访问，它可以允许多个线程同时访问资源，但访问的线程数量不能超过指定的许可数量。
综上所述，`CountDownLatch` 是一个非常实用的同步工具，它可以帮助我们实现线程间的等待和同步，提高程序的并发性能。 