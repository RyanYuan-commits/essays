## 什么是线程
进程：系统进行资源分配和调度的基本单位。
线程：进程的一个执行路径，进程中的多个线程共享进程的资源，线程是 CPU 分配的基本单位。
多个线程共享进程的堆和方法区
- 程序计数器：记录线程让出 CPU 时执行指令的地址，以备下次分配到 CPU 资源的时候可以继续执行
- 每个线程都有自己的栈资源，每个栈帧有如下的部分构成
    - 局部变量表：记录线程中使用到的变量
    - 操作数栈：执行指令的时候临时存放数据的位置
    - 栈数据：帧数据主要包含动态连接，方法出口，异常表的引用
## 线程的创建与运行
Java 中有三种线程创建的方式：分别为实现 Runnable 接口的 run 方法、继承 Thread 类并重写 run 方法，使用 FutureTask 的方式。
因为 Java 是单继承机制，当继承了 Thread 类就无法再去继承其他的类，实现 Runnable 接口就可以解决这个问题，而 FutureTask 有其他的应用场景
`FutureTask`主要应用于异步计算的场景。以下是一些具体的应用场景：
1. **异步计算**：当需要执行一个耗时的计算任务时，可以将该任务放入`FutureTask`，然后将`FutureTask`提交给`Executor`进行异步执行。这样，主线程可以在等待计算结果的同时，继续执行其他任务。当需要计算结果时，使用`FutureTask`的`get`方法获取。
2. **结果可获取的任务**：如果需要执行一个任务，并且需要获取任务的执行结果，那么可以使用`FutureTask`。因为`FutureTask`实现了`Future`接口，所以可以通过`Future`的`get`方法获取任务的执行结果。
3. **可取消的任务**：如果需要执行一个可以被取消的任务，那么可以使用`FutureTask`。`FutureTask`实现了`Future`接口，所以可以通过`Future`的`cancel`方法来取消任务的执行。
4. **多线程共享的计算结果**：如果有多个线程需要共享同一个计算结果，那么可以使用`FutureTask`。只需要将同一个`FutureTask`对象传递给这些线程，然后这些线程就可以通过`FutureTask`的`get`方法来获取共享的计算结果。
```java
public class _1_WaysOfCreateThread {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        MyThread1 thread1 = new MyThread1();
        Thread thread2 = new Thread(new MyThread2());
        FutureTask<String> stringFutureTask = new FutureTask<>(new CallerTask());
        Thread thread3 = new Thread(stringFutureTask);
        thread3.start();
        String s = stringFutureTask.get();
        System.out.println(s);
        thread1.start();
        thread2.start();
    }
    
    // 方式一
    static class MyThread1 extends Thread {
        @Override
        public void run() {
            System.out.println("方式一：继承Thread类");
        }
    }
    
    // 方式二
    static class MyThread2 implements Runnable {
        @Override
        public void run() {
            System.out.println("方式二：实现Runnable接口");
        }
    }
    
    // 方式三
    static class CallerTask implements Callable<String> {
        @Override
        public String call() throws Exception {
            System.out.println("进入");
            Thread.sleep(2000);
            return "方式三：FutureTask";
        }
    }
}
```

## 线程的等待与通知
### wait()方法
当一个线程调用了一个**共享变量**的时候，该调用线程会被阻塞挂起，知道发生了如下几件事情的时候才会继续执行：
	1）其他线程调用了该共享对象的 `notify()` 或者 `notifyAll()` 方法的时候
	2）其他线程调用了该线程的 `interrupt()` 方法的时候，线程会抛出 InterruptedException 并且返回。
线程获取监视器锁有如下的几种方式：
1）执行 synchronized 方法时候使用该共享变量作为参数
```java
synchronized(共享变量) {
	// doSomething
}
```
2）调用该共享变量的方法，且这个方法使用了 synchronized 修饰
```java
synchronized void add(int a, int b) {
	// doSomthing
}
```
**虚假休眠问题**
一个被挂起的线程有可能在没有条件使其苏醒或者中断的时候，自己被唤醒，这就是所谓的虚假唤醒；即使这种情况很少发生，但是防患于未然，我们可以在每次线程苏醒的时候去判断它是否满足苏醒的条件。
```java
synchronized(obj) {
	while(条件不满足) {
		obj.wait();
	}
}
```
通过这样的方式就可以实现即使在虚假苏醒的时候也能判断条件并且使得其再次进入挂起的状态。
```java
public class _2_ThreadNoticeAndWait {
    static final List<String> list = new ArrayList<>(10);

    static class Publisher extends Thread{
        @Override
        public void run() {
            synchronized (list) {
                while (!list.isEmpty()) {
                    try {
                        list.wait();
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                }
                while (true) {
                    while (list.size() < 10) {
                        try {
                            list.add(UUID.randomUUID().toString());
                            System.out.println("添加元素" + list.size());
                            Thread.sleep(500);
                        } catch (InterruptedException e) {
                            throw new RuntimeException(e);
                        }
                    }
                    list.notifyAll(); // 唤醒其他线程
                    try {
                        list.wait(); // 阻断并且让出自己的锁
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                }
            }
        }
    }

    static class Consumer extends Thread{
        @Override
        public void run() {
            synchronized (list) {
                while (list.isEmpty()) {
                    try {
                        list.wait();
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                }
                while (true) {
                    while (!list.isEmpty()) {
                        try {
                            System.out.println("获取内容" + list.removeLast());
                            Thread.sleep(500);
                        } catch (InterruptedException e) {
                            throw new RuntimeException(e);
                        }
                    }
                    list.notifyAll(); // 唤醒其他线程
                    try {
                        list.wait(); // 阻断并且让出自己的锁
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                }
            }
        }
    }

    public static void main(String[] args) {
        Publisher publisher = new Publisher();
        Consumer consumer = new Consumer();
        publisher.start();
        consumer.start();
    }
}
```
上面的方法展示了一个消息的发布者和处理者，发布者发布十条消息就通知消费者去消费和输出，然后消费完之后通知发布者生产，循环执行。
当线程使用 `wait()` 方法释放了共享对象的锁之后，**线程持有的其他的共享对象的监视器锁并不会被释放**，所以尽量不要再获取对象锁之后再去获取其他对象的锁，操作不当很容易造成死锁的情况。
### wait(long timeout)
该方法相比于前面的方法多了一个超时参数，这样的线程即使没有被其他线程唤醒或者中断，也会在满足超时时间之后自己结束休眠。
注意 `wait(0)` 的效果和 `wait()` 的效果是相同的，实际上 `wait()` 方法就是直接调用了 `wait(0)`
`void wait()`
```java
    public final void wait() throws InterruptedException {
        wait(0L);
    }
```
### wait(long timeout, int nanos)
```java
    public final void wait(long timeoutMillis, int nanos) throws InterruptedException {
        if (timeoutMillis < 0) {
            throw new IllegalArgumentException("timeoutMillis value is negative");
        }

        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }

        if (nanos > 0 && timeoutMillis < Long.MAX_VALUE) {
            timeoutMillis++;
        }

        wait(timeoutMillis);
    }
```

此方法的参数有两个，分别是毫秒数（timeoutMillis）和纳秒数（nanos）。timeoutMillis参数定义了线程等待唤醒的最长时间，单位是毫秒。nanos参数是额外的等待时间，单位是纳秒，取值范围在0到999999。如果 `nanos` 大于0，且 `timeoutMillis` 小于 `Long.MAX_VALUE`，那么实际的等待时间会比指定的超时时间（timeoutMillis）多1毫秒，以确保总的等待时间不会少于指定的时间。这主要是因为计算机系统中的时间测量和表示通常不能精确到纳秒级别。
### notify() 方法
一个线程调用了共享对象的 `notify()` 方法之后，会唤醒一个在该共享变量上调用 `wait()` 系列方法后被挂起的线程。一个共享变量上可能会有多个线程在等待，具体唤醒哪个等待的线程是随机的。
此外，被唤醒的线程也不是马上就继续执行的，它必须获取共享对象的监视器锁之后才能继续只能怪，也就是唤醒它的线程释放锁之后。
和 `wait()` 系列方法相同，这个方法也是只有在获取了监视器锁之后才能执行，否则会抛出 IllegalMonitorStateException 异常。
### notifyAll() 方法
与上面不同的是，这个方法会唤醒所有被阻塞到该共享变量上的线程。
## 等待线程执行终止的 join 方法
`join()`方法是一个实例方法，主要作用是让调用这个方法的线程等待调用`join()`方法的线程终止后才能继续执行。如果线程已经终止，`join()`方法立即返回。这个方法允许一个线程强制运行另一个线程，直到其运行完成。例如，如果线程B在线程A中调用`b.join()`，那么线程A将会被阻塞，直到线程B完成执行。
`join()`方法也有带超时参数的版本，它们允许等待一定的时间，而不是无限期地等待。例如，`join(2000)`会等待最多2000毫秒的时间，不管线程是否实际完成。
## 让线程休眠的 sleep 方法
在一个线程中执行了这个方法后，线程会暂时让出指定时间的执行权，也就是在这段时间内不参与 CPU 的调度，**但是该线程持有的监视器资源，比如监视器锁是继续持有而不让出的**
## 让出 CPU 执行权的 yield 方法
yield()方法是一个与调度相关的静态方法，调用它可以让当前线程主动让出CPU的执行权，使得线程调度器可以重新调度线程。
换句话说，yield()方法可以让当前线程从"运行状态"退回到"就绪状态"，以使得其他相同优先级的等待线程得到执行的机会。
然而，需要注意的是，yield()方法**并不能保证使得当前线程立即停止执行**，因为**线程调度器可能会再次将执行权分配给刚刚让出执行权的线程**。
```java
    static class YieldThread extends Thread{
        @Override
        public void run() {
            for (int i = 0; i < 2; i++) {
                System.out.println(Thread.currentThread().getName() + "try to yield");
                Thread.yield();
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
            System.out.println(Thread.currentThread().getName() + "finish work");
        }
    }

    public static void main(String[] args) {
//        Publisher publisher = new Publisher();
//        Consumer consumer = new Consumer();
//        publisher.start();
//        consumer.start();
        YieldThread yieldThread1 = new YieldThread();
        YieldThread yieldThread2 = new YieldThread();
        YieldThread yieldThread3 = new YieldThread();
        yieldThread1.start();
        yieldThread2.start();
        yieldThread3.start();
    }
```
## 1.7）线程中断
Java 中的线程中断是线程间的**协作模式**，通过设置线程的通断标志并不能直接阻止该线程的执行，而是被终端的线程根据中断的状态自行去处理。
- void interrupt() 方法：终端线程，当线程 A 运行的时候，线程 B 调用线程 A 的这个方法可以设置 A 的中断标志为 true，但是 A 实际上是并没有被中断的，会继续执行，但是当调用 wait、join、sleep 方法被挂起的时候，此时调用就会导致 A 会在调用这些方法的地方抛出 InterruptedException 异常而返回。
- boolean isInterrupted() 方法：检测当前的线程是否被中断，如果是的话就返回 true，否则就返回 false。
```java
    
    public boolean isInterrupted() {
        return isInterrupted(false);
    }
    
    // 底层实际调用的是 native 方法，其中可以指定是否要清除被中断状态
    private native boolean isInterrupted(boolean ClearInterrupted);
```
- boolean interrupteed() 方法：检测当前的线程是否被中断，如果是返回 true，反之则返回 false；与上面的方法不同的是，如果发现当前线程被终端，则会清除终端的标志，方法是 static 方法，通过 Thread 类来直接调用
```java
    public static boolean interrupted() {
        return currentThread().isInterrupted(true); // 此时是 true，isInterrupted 方法只有这两个地方被调用
    }
```
## 线程上下文切换
在多线程编程中线程的个数一般是都大于 CPU 的个数的，而 CPU 同一个时刻之中只能被一个线程使用，但是用户实际上是感觉在同时执行（并发），实际上就是因为 CPU 资源的分配采用了时间片轮转的策略，也就是给每个县城分配一个时间片，线程在时间片内占用 CPU 执行任务。当前线程使用完时间片之后，就会处于就绪的状态并让出 CPU 让其他线程占用，这就是上下文切换，从当前线程的上下文切换到了其他线程。此时就需要存储让出 CPU 的时候，线程执行到了哪一个步骤，以便于下次切换到这个线程的时候可以快速的恢复。
## 线程死锁
### 什么是线程死锁？
死锁指的是在两个或者两个以上的线程在执行的过程中，因为彼此争夺资源而造成的相互等待的现象，在没有外力作用的情况下，这些线程会**一直等待**而无法继续运行下去。

死锁的产生需要以下的四个条件
- 互斥条件：线程对已经获取到的资源进行排他性使用，即资源只能由一个线程占用。如果其他线程请求这个资源，那其他的线程只能等待。
- 请求并持有条件：一个线程持有了一个资源，还去请求其他的资源，而新的资源又被其他线程占有，当前线程就会被阻塞，且阻塞的同时不会释放自己获取的资源
- 不可剥夺条件：一个资源被线程使用完之前不会被其他线程占用
- 环路等待条件：在发生死锁的时候，必然存在一个线程 - 资源的环形链路，即线程 0 在等待线程 1 资源，1 在等待 2…… n 在等待线程 0 的资源。
线程 A 首先获取到了 resourceA 中的锁资源，然后请求等待 resourceB 上的锁资源，这就构成了 **请求并持有条件**
线程 A 持有了 resourceA 之后又去请求 线程 B 持有的 resourceB 资源，这就构成了 **环路等待条件**
### 如何避免线程死锁？
想要避免线程死锁，只需要破坏四个条件中的一个即可，但是其中仅有两个条件是可以被破坏的：
- 请求并持有条件
- 环路等待条件
这两个条件可以通过调整 申请资源的顺序 来破坏，线程 A 和线程 B 都需要得到这两个 resource 资源，但是它们的请求顺序是相反的，但如果我们将请求资源顺序设置成有序的，也就是 **资源申请的有序性原则**。
```java
/**
 * 测试线程死锁
 */
public class DeadLock {
    static final Object sourceA = new Object();
    static final Object sourceB = new Object();

    public static void main(String[] args) {
        Thread threadA = new Thread(() -> {
            System.out.println("threadA try to get sourceA");
            synchronized (sourceA) {
                try {
                    Thread.sleep(2000);
                    System.out.println("threadA try to get sourceB");
                    synchronized (sourceB) {}
                    System.out.println("threadA finished");
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        });
        Thread threadB = new Thread(() -> {
            System.out.println("threadB try to get sourceA");
            synchronized (sourceA) {
                try {
                    Thread.sleep(3000);
                    System.out.println("threadB try to get sourceB");
                    synchronized (sourceB) {}
                    System.out.println("threadB finished");
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        });
        threadA.start();
        threadB.start();
    }
}
```
线程的有序性就是指假如线程 A 和线程 B 都需要资源 1 2 3……n 的时候，对资源进行排序，线程 A 和线程 B 只有在获取了资源 n - 1 之后才能获取资源 n。
线程的有序性可以避免死锁，比如上面的代码中，线程 A 和线程 B 都去请求资源 A，但只有一个线程可以请求到，此时另一个线程会被阻塞，而不会再去请求资源 B，这样就破坏了请求持有条件和环路等待条件。
## 守护线程和用户线程
Java 中的线程分为两类，分别为 deamon 线程（守护线程）和 user 线程（用户线程），在 JVM 启动的时候会调用 main 函数，main 函数所在的线程就是用户线程，其实 JVM 同时还启动了很多守护线程，比如垃圾回收线程。
守护线程和用户线程的区别就是当最后一个非守护线程结束之后，整个程序就会结束，不管现在的守护线程是否结束。
将线程设置为守护线程也很简单，只需要通过调用线程的 setDeamon(true) 方法，将线程的 deamon 参数设置为 true 即可。
```java
public class DaemonTest {
    public static void main(String[] args) {
        Thread daemonThread = new Thread(() -> {
            while (true) {}
        });
        Thread userThread = new Thread(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            System.out.println("userThread finished");
        });
        daemonThread.setDaemon(true);
        daemonThread.start();
        userThread.start();
    }
}
```
## 锁的概述

### 乐观锁与悲观锁
**悲观锁**：悲观锁对于数据对外界的修改保持保守态度，即认为数据很容易被其他线程修改，所以会在数据被修改之前对数据进行加锁的操作，并在整个修改的过程中，使数据处于锁定的状态。悲观锁一般使用的就是数据库自带的锁。
**乐观锁**：乐观锁是一种用于并发控制的机制，它的核心思想是乐观地认为在执行更新操作时不会发生冲突。与悲观锁相反，乐观锁不会在开始执行操作之前加锁，而是在执行操作后再检查是否发生了冲突。
乐观锁通常涉及以下几个步骤：
1. **读取数据：** 首先，获取要更新的数据，并记录下数据的版本号或者其他标识符。
2. **执行操作：** 在获取数据后，执行更新操作，这包括修改数据的值或者执行其他操作。
3. **检查冲突：** 在更新操作执行完成后，乐观地假设在执行操作时没有其他线程对数据进行了修改。因此，乐观锁会重新读取数据，并比较之前记录的版本号或标识符与当前数据的版本号或标识符是否一致。
4. **处理冲突：** 如果发现数据已经被其他线程修改，说明发生了冲突。在这种情况下，可以选择重试更新操作、回滚操作、抛出异常等方式来处理冲突。
乐观锁通常基于数据的版本控制或时间戳来实现。每次更新操作都会增加数据的版本号或者更新时间戳，这样可以通过比较版本号或时间戳来检测数据是否发生了变化。如果数据没有发生变化，则认为操作是成功的；如果数据发生了变化，则需要进行相应的处理。
乐观锁适用于读操作频繁、写操作较少、冲突概率低的场景，因为它不会在执行操作之前加锁，可以降低锁的竞争和开销，提高并发性能。常见的应用场景包括数据库的乐观并发控制、版本控制系统等。
### 公平锁和非公平锁
根据线程获取锁的前瞻机制，锁可以分为公平锁和非公平锁。
公平锁表示线程获取锁顺序是按照线程请求锁的时间早晚来决定的，也就是最早请求的线程一定最先获取到锁。而非公平锁则在运行的时候闯入，也就是先来的不一定先得。
在 **`ReentrantLock`** 中，默认情况下是非公平锁，即构造函数不传入参数时，创建的是非公平锁。如果需要创建公平锁，可以通过在构造函数中传入 **`true`** 来指定公平性，例如：
```java
ReentrantLock fairLock = new ReentrantLock(true); // 创建公平锁
```
在公平锁中，线程的获取顺序与它们请求锁的顺序一致，因此可以避免某些线程长时间等待而导致的饥饿现象。然而，公平锁可能会降低并发性能，因为每次获取锁时都需要进行等待队列的遍历，确定是否有等待的线程。
### 独占锁与共享锁
根据锁是否能被多个线程持有，分为共享锁和独占锁。
独占锁保证任何时候只有一个线程可以得到锁；共享锁则可以由多个线程去持有，比如 ReadWriteLocal 读写锁，它允许一个资源可以被多线程同时进行读的操作。
### 可重入锁
当一个线程要获取被其他线程持有的独占锁的时候，线程会阻塞，但当一个线程获取自己已经获取的锁的时候是否会被阻塞呢？如果不被阻塞，那就可以说这个锁是可重入锁，只要线程获取到了锁，就可以无限（严格来说是有限）次进入被锁锁住的代码。
```java
class Hello {
    public synchronized void helloA() {
        System.out.println("A");
    }

    public synchronized void helloB() {
        System.out.println("B");
    }
}
```
线程先去调用 helloB 之前会获取内置锁，之后打印输出，然后调用 helloA 方法，在调用之前也需要获取锁，但是如果锁不是可重入的就会一直被阻塞。
synchronized 锁是可重入锁，可重入锁的原理是在锁的内部维护一个线程的标示，用来标示锁被哪个线程占用，当一个线程获取到了锁的时候，计数器的值会变成 1，这时候其他线程获取锁的时候会发现锁所有者不是自己，而被挂起。
但是当发现锁是属于自己的，就是将计数器加一，当释放锁的时候，计数器减一，当计数器为 0 的时候，锁的线程被标示被重置为 null，这时候其他线程就会开始竞争这个锁。
### 自旋锁
自旋锁是一种用于多线程编程中的锁机制，它采用“**忙等待**”的方式来实现线程的同步。当一个线程尝试获取锁时，如果发现锁已经被其他线程占用，则该线程会循环执行一段无意义的忙等待代码，直到获取到锁为止。
自旋锁的主要特点包括：
1. **忙等待：** 当一个线程尝试获取锁时，如果发现锁已经被其他线程占用，则该线程会循环执行一段无意义的忙等待代码，直到获取到锁为止。这种方式避免了线程在等待锁时进入阻塞状态，从而减少了线程切换和上下文切换的开销。
2. **无阻塞：** 自旋锁是一种无阻塞的同步机制，因为线程在等待锁时不会被阻塞，而是继续执行忙等待代码。这种方式可以避免线程在等待锁时进入阻塞状态，从而提高了程序的响应速度和并发性能。
3. **短暂持有：** 自旋锁通常用于保护临界区代码，而临界区代码的执行时间应该尽可能短暂，以减少其他线程的等待时间和忙等待的开销。
4. **适用性：** 自旋锁适用于在多核处理器上运行的并发程序，因为它可以充分利用处理器的并发执行能力。在单核处理器上运行的并发程序通常不适合使用自旋锁，因为它可能会导致线程长时间的忙等待，浪费了处理器资源。
自旋锁在一些场景下可以提供较好的性能，特别是在临界区代码执行时间较短、线程竞争较激烈的情况下。然而，自旋锁也存在一些缺点，例如忙等待可能会消耗大量的处理器资源，如果锁被长时间占用，会导致其他线程长时间的忙等待，影响程序的性能。因此，自旋锁的使用需要根据具体的场景来评估和选择。