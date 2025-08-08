> `CyclicBarrier` 是 Java 并发包 `java.util.concurrent` 中一个强大的同步工具，它允许一组线程相互等待，直到所有线程都到达一个公共的屏障点，之后所有线程才会继续执行后续操作。由于这个屏障点在所有等待线程释放后可以被重用，所以被称为循环屏障。
### 1 基本概念
`CyclicBarrier` 内部维护一个计数器，该计数器的初始值是在创建 `CyclicBarrier` 对象时指定的，代表需要等待的线程数量。当一个线程调用 `await()` 方法时，它会被阻塞，直到所有线程都调用了 `await()` 方法，此时计数器的值变为 0，所有被阻塞的线程将被唤醒，继续执行后续操作。同时，`CyclicBarrier` 还可以指定一个可选的屏障操作，该操作会在所有线程到达屏障点后，在所有线程继续执行之前由最后一个到达的线程执行。
### 2 使用方法
#### 2.1 构造方法
`CyclicBarrier` 有两种构造方法：
```java
// 创建一个新的 CyclicBarrier，它将在给定数量的参与者（线程）都到达屏障后启动
public CyclicBarrier(int parties)

// 创建一个新的 CyclicBarrier，它将在给定数量的参与者（线程）都到达屏障后启动，并在所有线程释放屏障前执行给定的屏障操作
public CyclicBarrier(int parties, Runnable barrierAction)
```
其中，`parties` 表示需要等待的线程数量，`barrierAction` 是一个 `Runnable` 对象，表示屏障操作。
#### 2.2 主要方法
- **`int await()`**：使当前线程等待，直到所有线程都到达屏障点。如果当前线程不是最后一个到达的线程，则会被阻塞。当所有线程都到达屏障点后，该方法会返回当前线程在到达屏障点时的索引（从 0 开始）。如果在等待过程中当前线程被中断，会抛出 `InterruptedException` 异常；如果屏障被破坏，会抛出 `BrokenBarrierException` 异常。
- **`int await(long timeout, TimeUnit unit)`**：使当前线程等待，直到所有线程都到达屏障点或者等待时间超过指定的时长。如果在等待时间内所有线程都到达了屏障点，则该方法会返回当前线程在到达屏障点时的索引；否则返回 -1。同样，如果在等待过程中当前线程被中断，会抛出 `InterruptedException` 异常；如果屏障被破坏，会抛出 `BrokenBarrierException` 异常。
- **`int getParties()`**：返回需要等待的线程数量。
- **`boolean isBroken()`**：查询屏障是否被破坏。如果有线程在等待过程中被中断、超时或者调用了 `reset()` 方法，则屏障会被破坏。
- **`void reset()`**：将屏障重置为初始状态。如果有线程正在等待，则这些线程会抛出 `BrokenBarrierException` 异常。
#### 2.3 示例代码
```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class CyclicBarrierExample {
    public static void main(String[] args) {
        // 创建一个 CyclicBarrier 实例，指定等待的线程数量为 3，并指定屏障操作
        CyclicBarrier barrier = new CyclicBarrier(3, () -> {
            System.out.println("所有线程都已到达屏障点，执行屏障操作");
        });

        // 创建并启动 3 个线程
        for (int i = 0; i < 3; i++) {
            new Thread(new Worker(barrier)).start();
        }
    }

    static class Worker implements Runnable {
        private final CyclicBarrier barrier;

        public Worker(CyclicBarrier barrier) {
            this.barrier = barrier;
        }

        @Override
        public void run() {
            try {
                System.out.println(Thread.currentThread().getName() + " 正在执行任务");
                // 模拟任务执行
                Thread.sleep((long) (Math.random() * 1000));
                System.out.println(Thread.currentThread().getName() + " 到达屏障点");
                // 线程到达屏障点，等待其他线程
                int index = barrier.await();
                System.out.println(Thread.currentThread().getName() + " 所有线程都已到达屏障点，继续执行后续任务，索引为: " + index);
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
    }
}
```
- 在 `main` 方法中，创建了一个 `CyclicBarrier` 实例，指定需要等待的线程数量为 3，并指定了一个屏障操作。
- 启动 3 个线程，每个线程执行 `Worker` 类的 `run` 方法。
- 在 `run` 方法中，线程先模拟执行任务，然后调用 `barrier.await()` 方法到达屏障点并等待其他线程。
- 当所有 3 个线程都调用了 `await()` 方法后，最后一个到达的线程会执行屏障操作，然后所有线程将同时继续执行后续任务。
### 3 实现原理
`CyclicBarrier` 是基于 `ReentrantLock` 和 `Condition` 实现的。它内部维护了一个计数器和一个 `Generation` 对象，用于表示当前的屏障代。当一个线程调用 `await()` 方法时，会先获取锁，然后将计数器减 1。如果计数器不为 0，则线程会在 `Condition` 上等待；如果计数器为 0，则会执行屏障操作，然后唤醒所有等待的线程，并重置计数器和 `Generation` 对象，以便下一轮使用。
### 4 应用场景
- **并行计算**：在进行大规模数据处理时，可以将数据分成多个部分，每个线程负责处理一部分数据。当所有线程都完成自己的任务后，使用 `CyclicBarrier` 进行同步，然后对所有线程的处理结果进行汇总。
- **多线程游戏**：在多线程游戏中，每个线程可能负责控制一个角色的行为。在每个游戏回合开始前，使用 `CyclicBarrier` 确保所有角色都准备好，然后同时开始新的回合。
- **分阶段任务**：对于一些需要分阶段执行的任务，每个阶段都需要所有线程完成该阶段的任务后才能进入下一阶段。可以在每个阶段的末尾使用 `CyclicBarrier` 进行同步。
### 5 优缺点
优点
- **可重用性**：`CyclicBarrier` 可以被多次使用，在一次同步完成后，它会自动重置，等待下一轮线程的同步。
- **支持屏障操作**：可以指定一个屏障操作，在所有线程到达屏障点后执行，方便进行一些汇总或初始化操作。
- **公平性**：`CyclicBarrier` 是公平的，线程按照到达屏障点的顺序被唤醒。
缺点
- **容易被破坏**：如果有线程在等待过程中被中断、超时或者调用了 `reset()` 方法，则屏障会被破坏，后续调用 `await()` 方法的线程会抛出 `BrokenBarrierException` 异常。
- **性能问题**：在高并发场景下，频繁的加锁和解锁操作可能会导致性能下降。
### 6 与其他并发工具的对比
- **与 `CountDownLatch` 的对比**：
  - `CountDownLatch` 是一次性的，计数器的值只能减不能增，一旦计数器变为 0，就不能再使用；而 `CyclicBarrier` 是可重用的，当所有线程都通过屏障后，它可以被再次使用。
  - `CountDownLatch` 主要用于一个或多个线程等待其他线程完成任务；而 `CyclicBarrier` 主要用于多个线程之间互相等待，直到所有线程都到达某个公共屏障点。
- **与 `Semaphore` 的对比**：
  - `CyclicBarrier` 主要用于线程间的同步，确保所有线程都到达某个点后再继续执行；而 `Semaphore` 主要用于控制对有限资源的访问，它可以允许多个线程同时访问资源，但访问的线程数量不能超过指定的许可数量。