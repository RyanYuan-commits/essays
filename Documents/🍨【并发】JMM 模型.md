JMM 是一个非常抽象的概念，它全称是 Java Memory Model（java 内存模型），听起来很像是在描述 JVM 的内存区域，但实际上，它是对多线程环境下关键问题的解决方式的一种 **抽象**。
## 多线程环境下有哪些关键问题？
1）可见性问题：一个线程对共享变量的修改，如何让其他线程能立即看到。
2）有序性问题：不同线程对共享变量的读写操作如何按合理的顺序进行，避免指令重排序带来的问题。
3）原子性问题：确保多个线程之间某些操作不可分割，避免出现竞争条件（race condition）。
直接搬出这些定义显然是没有任何体会的，我们直接来看个例子，来对上面的问题有一个更加直观的体会。
```java
/**  
 * 演示可见性问题导致的死循环
 */
public class Main {  
    static boolean stop = false;  
    public static void main(String[] args) throws InterruptedException {  
        Thread writerThread = new Thread(() -> {  
            while (!stop) {}  
            System.out.println("等待线程停止");  
        });  
        writerThread.start();  
        Thread.sleep(1000); // 等待1秒  
        // 修改共享变量  
        stop = true;  
        System.out.println("主线程将 stop 变量设置为 true");  
    }  
}
```
当执行上面的代码的时候，有可能线程 2  会永远不会退出，即使主线程已经将 `stop` 变更为 `true`。那么为什么会出现这种情况呢？
![[双核 CPU 的系统架构简图.png|700]]
上图展示的是一个双核 CPU 的系统架构，每个核心都有自己的控制器和运算器。
通过上图可以看出来，每个核心都有自己的一个一级缓存（L1 Cache），在这个下面还有一个所有 CPU 都共享的一个二级缓存（L2 Cache）。
当一个线程操作共享变量时，它首先 **从主内存复制共享变量到自己的工作内存**，然后对工作内存里的变量进行处理，处理完后将变量值更新到主内存。
那么，假如线程 A 和线程 B 同时处理一个共享变量，会出现什么情况呢？假设两个线程使用不同的 CPU 执行，并且开始的时候它们的一级缓存都为空。共享变量的值为 0，两个线程对共享变量都进行自增的操作。
1. 线程 A 首先尝试去获取共享变量的值，由于两个缓存都没有命中，线程 A 会去主内存中去获取值，并且将这个值缓存到两个缓存中；此时线程 A 将共享变量设置为 1吗，并且写回线程 A 的一级、二级缓存，以及主内存中；**此时线程 A 的两级缓存和主内存中的值都为 1**。
2. 线程 B 的两级缓存未命中，将主内存中的共享变量取出来放到自己的缓存中，此时的共享变量由于线程 A 的修改，已经变为了 1，线程 B 在其基础上将其修改为 2，并且刷新到自己的两级缓存和主内存中；此时，整个过程是没有出现问题的。
3. 随后，线程 A 想对共享变量继续操作，读取一级缓存成功了，此时 A 获取到的共享变量的值就是 1；
==到这里，问题就出现了，线程 B 已经将变量修改为了  2，为什么线程 A 读取到的值还是 1 呢==？ 这就是上面提出的可见性问题，线程 B 的修改对线程 A 来说是不可见的。
比起上面具体到 CPU 的图，大家可能对 **直接内存** 和 **主内存** 两个词更加熟悉
![[直接内存与主内存.png|600]]
并不是所有的 CPU 都有二级缓存，但基本上所有线程的操作，都可以用上面这个 直接内存 和 主内存 的抽象来讲述清楚。
这就是 JMM 做的第一层抽象，**JMM 将内存划分为主内存和直接内存（工作内存），通过这两者的抽象，解释了线程如何通过工作内存与主内存进行数据交互**。
在上面的案例中，由于等待线程从自己的缓存中获取对象，导致了其一直在等待，而如果我们用 `volatile` 来修饰变量就不会出现这种情况了；因为被 `volatile` 修饰的变量，无论是读或者写，都会从主内存中进行，修改会立刻被刷回主内存，这样就保证了可见性。
将变量改为 `volatile` 后，Waiter 线程就正常退出了。
## volatile、synchronized
我们继续讲完剩下的两个多线程问题，也就是有序性问题和原子性问题。
对于这两个问题，可以举一个单例模式双重检查锁的案例，双重检查锁的存在就是为了解决这两个问题：
```java
public class Singleton_05 { 
	private static volatile Singleton_05 instance; 
	public static Singleton_05 getInstance() { 
		if (null != instance) return instance; 
		synchronized (Singleton_05.class) { 
			if (null == instance) { 
				instance = new Singleton_05();
			} 
		} 
		return instance; 
	} 
}
```
上面展示的是单例模式懒汉式的代码，并且使用双重检查锁来保证了线程安全性；
创建对象的步骤其实是一个非原子性操作，通常包括以下几步：
1. 分配内存空间；
2. 调用构造方法，初始化对象；
3. 将对象的引用指向分配的内存空间（将这个对象的引用赋值给 instance 变量）。
synchronized 保证了创建整个对象的操作是不可分割的，它确保在同一时刻，只有一个线程可以进入临界区（即创建实例的代码块），防止多个线程同时创建实例。
但是由于处理器和编译器有可能会对上面的三条指令进行重排序，比如将第二步和第三步颠倒过来，这会导致在多线程的环境下，其他线程拿到的不是一个完整的对象，这样还需要有一个关键字来保证上述三个操作不会由于重排序而导致先后顺序的变化，而 private static volatile Singleton_05 instance 中的 volatile 关键字就可以保证它修饰的变量不会被指令重排序影响，volatile 变量的读写操作不能都不能与其之前的操作重排序。
volatile 防止指令重排序的原理是这样的：JVM 会在该变量的写操作之前插入一个写屏障，在该变量的读操作之后插入一个读屏障。这意味着，volatile 变量的写操作不能与其之前的操作重排序，volatile 变量的读操作也不能与其之后的操作重排序。
- Load Barrier（读屏障）：确保在读取 v 之后，所有后续操作都会使用从主内存读取的最新值，而不会读取工作内存的过期数据。
- Store Barrier（写屏障）：确保在 v 写入前，所有之前的普通变量写操作（若有）都已经刷新到主内存；确保 v 的写入操作立即对其他线程可见。
同时，写屏障 volatile 也保证了对变量操作的可见性。这两个关键字就是 JMM 的第二层抽象，我们都知道解决上面的问题需要保证 原子性、可见性 和 顺序性，那如何保证呢？
JMM 就为我们提供了 volatile、synchronized 等关键字，这些关键字是 JMM 规则在 Java 语言中的具体表现，它们提供了线程安全的基本工具，我们不用在意系统底层的细节，只需要通过这些关键字，就能在上面 直接内存 和 主内存 结构模型的基础上，完成多线程程序的设计和编写。
## Happens-Before 原则
==Happens-before== 规则 是 Java 内存模型（JMM）中的一组规则，用来确保多线程环境下的操作顺序，解决**可见性**和**重排序**问题；
我们平时的一些常识其实是这一组规则中的某一条规则；这些规则定义了哪些操作必须发生在其他操作之前，以确保线程间的内存一致性和正确的执行顺序。

1）程序顺序规则：在一个线程中，程序中的每个操作**按照程序的顺序**发生，先写的操作 **happens-before** 后写的操作。这条规则适用于单线程，即单线程内的操作按照代码编写的顺序发生。
   ```java
   int a = 1; // 操作 A
   int b = 2; // 操作 B
   ```
操作 A happens-before 操作 B。
   
2）监视器锁规则（synchronized 规则）：对一个锁的解锁操作，unlock，happens-before 于对同一个锁的后续加锁操作之后，lock。这意味着，在某个线程释放锁之后，其他线程在获取这个锁时能够看到之前线程在临界区内的修改。
```java
synchronized(lock) { // 线程 A 解锁 happens-before 线程 B 加锁
	// 操作 A
}
synchronized(lock) {
	// 操作 B
}
```
   线程 A 的解锁 happens-before 线程 B 的加锁。
   
3）volatile 变量规则：对 volatile 变量的写操作，happens-before 于后续对该 volatile 变量的读操作。
这确保了当一个线程写入 volatile 变量后，其他线程能够立即看到这个写入操作的结果。
```java
volatile int x;
x = 1; // 写 happens-before
int y = x; // 读
```
`x = 1` 的写操作 happens-before 后面的 y = x 的读操作。
   
4）线程启动规则：在线程 Thread.start() 调用之前发生的所有操作，happens-before 于该新线程的任何操作。这意味着，在调用 `start()` 之前，主线程的所有写操作对新启动的线程可见。
```java
Thread t = new Thread(() -> {
	// 操作 B
});
	// 操作 A happens-before 线程启动后的操作 B
t.start();
```
主线程的操作 A happens-before 新线程中的操作 B。

5）线程终止规则：线程内的所有操作，happens-before 另一个线程调用 Thread.join() 并成功返回。这意味着，当主线程调用 join() 并等待子线程结束时，子线程中的所有操作对主线程可见。
   ```java
Thread t = new Thread(() -> {
   // 操作 A
});
t.start();
t.join(); // 操作 A happens-before 主线程 join 之后的操作
// 操作 B
   ```
子线程的操作 A happens-before 主线程中的操作 B（在 join() 之后）。

6）线程中断规则：当一个线程调用 interrupt() 中断另一个线程时，interrupt() 的调用 happens-before 该线程检测到中断信号（即 Thread.interrupted() 或抛出 InterruptedException）。
   ```java
Thread t = new Thread(() -> {
   try {
	   Thread.sleep(1000);
   } catch (InterruptedException e) {
	   // 检测到中断
   }
});
t.start();
t.interrupt(); // 中断调用 happens-before 中断被检测
   ```

8）对象终结规则：对象的构造函数执行完毕 happens-before 该对象的 `finalize()` 方法的调用。这确保在对象的终结方法执行前，所有构造操作已完成。

9）传递性：如果操作 A happens-before 操作 B，且操作 B happens-before 操作 C，那么操作 A happens-before 操作 C。
   ```java
操作 A happens-before 操作 B;
操作 B happens-before 操作 C;
   ```
根据传递性，操作 A happens-before 操作 C。
## synchronized 的实现原理
synchronized 关键字底层属于 JVM 层面，它为什么重量级、它的锁升级机制到底是什么，都是经常会问到的问题。先来看一段代码：
```java
public class Main {  
    static int a = 0;  
    public static void main(String[] args) throws InterruptedException {  
        synchronized (Main.class) {  
            a += 1;  
        }  
    }  
}
```
它编译后的字节码是这样的：
```java
 0 ldc #2 <com/ryan/Main>
 2 dup
 3 astore_1
 4 monitorenter
 5 getstatic #3 <com/ryan/Main.a : I>
 8 iconst_1
 9 iadd
10 putstatic #3 <com/ryan/Main.a : I>
13 aload_1
14 monitorexit
15 goto 23 (+8)
18 astore_2
19 aload_1
20 monitorexit
21 aload_2
22 athrow
23 return
```
我们关注这么两条指令：`monitorenter` 和 `monitorexit`，其中 `monitorenter` 指令指向同步代码块的开始位置，`monitorexit` 指令则指明同步代码块的结束位置。
至于为什么有两个 `monitorexit`，可以看到，第一个 `monitorexit` 的下一条语句是 `goto 23` 也就是直接到 `return`，而第二个 `monitorexit` 之后的是 `athrow`，所以第二个 `monitorexit` 处理的是抛出异常的情况。
如果方法正常进行，会在第一个 `monitorexit` 释放锁后到达 `return`，而抛出异常，则就会执行第二个 `monitorexit` 周围的语句。
这两个 `monitorexit` 保证了无论是否抛出异常，锁都会被释放。
上面提到，被 `synchronized` 修饰的代码块或方法中的内容会被 `monitorenter` 和 `monitorexit` 包裹，那什么是 `monitor` 结构呢？
![[synchronized架构.png]]

上图展示是一个 monitor 的基本结构，它由这三部分构成：
1. Entry Set：等待获取对象锁的线程。
2. The Owner：获取到锁的线程（只能有一个）,获取到锁的线程可以通过 `notify` 等方法来唤醒其他线程。
3. Wait Set：等待的线程，比如在 `synchronized` 代码块中调用 `wait` 方法的线程。
但是 `monitor` 依赖的是操作系统级别的 mutex lock，使用它会导致用户态和内核态的切换，带来很大的性能开销。
所以在 JDK6 之后，引入了锁升级的机制：
`synchronized` 是依赖对象头中的 Mark World 来实现的，里面会含有对象的锁信息：
![[对象头中对锁状态的存储.png|700]]
锁分为以下的四种状态：
1）==无锁==：没有对资源进行锁定，所有线程都可以对资源进行访问；适用于这个变量不会出现在多线程的环境下，或者即使出现在多线程的环境下，但我不想对这个变量进行保护的情况，或者使用无锁编程的方式来修改这个变量。
2）==偏向锁==：一个对象被加锁了，但是实际上只会有一个线程去获取锁，此时就会采用偏向锁的方式去优化，对象头中存储的是线程 ID，当线程需要获取锁的时候，检查线程的 ID 是否为对象头中存储的 ID，如果是的话，就可以直接让线程获取到锁。
3）==轻量级锁==：当对象处于偏向锁的状态，而此时有另外的线程进行竞争的话，轻量级锁会升级为偏向锁，此时对象头中存储的是指向栈中锁的记录的指针。
当线程通过 CAS 获取到锁之后，会存储一份线程对象头的副本到自己的方法栈中的 Lock Record 中，同时，这个区域中会有一个 owner 指针指向对象。
对象中又存有一个指向 Lock Record 中锁信息的指针，这样，锁和线程都知道了对方的存在；此时再有其他线程希望获取这个对象锁的时候，就会轮询去尝试获取锁。
自旋操作相比于 CPU 调度，在能较快获取到锁的场景下会有极好的表现。
4）==重量级锁==：一旦等待的线程超过一个，轻量级锁会被升级为重量级锁，它的实现就是我们前面提到的 mutex lock，是依赖操作系统实现的监视器锁。

---

JMM（Java 内存模型）可以看作是针对多线程环境下内存可见性、有序性和一致性问题的一种抽象解决方案。
它对如下部分做了抽象：
- 主内存与工作内存的抽象：JMM 将内存划分为主内存（共享内存）和工作内存（线程私有的缓存），通过这两者的抽象，解释了线程如何通过工作内存与主内存进行数据交互。
- volatile、synchronized等具体实现：这些关键字是 JMM 规则在 Java 语言中的具体表现，它们提供了线程安全的基本工具，用于确保多线程中的变量可见性、操作的有序性和原子性。
-  happens-before 规则的抽象：JMM 提供了一套 happens-before 规则，这是一组抽象的时序关系，用来说明在多线程环境下，某些操作在时间顺序上应该如何排列，以保证线程间操作的正确性和有序性。
