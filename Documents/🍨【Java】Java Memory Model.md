---
type: Java 基础
finished: "true"
---
## 1 多线程环境下的可见性问题

### 1.1 案例代码

是指: 一个线程对共享变量的修改, 其他线程无法立刻感知.

```java
/**  
 * 演示可见性问题导致的死循环
 */
public class Main {  
    static boolean stop = false;  
    public static void main(String[] args) throws InterruptedException {  
        Thread waiter = new Thread(() -> {  
            while (!stop) {}  
            System.out.println("等待线程停止");  
        });  
        waiter.start();  
        Thread.sleep(1000); // 等待1秒  
        // 修改共享变量  
        stop = true;  
        System.out.println("主线程将 stop 变量设置为 true");  
    }  
}
```

- 创建一个等待者线程, 不断读取 `stop` 的值, 当 `stop` 变为 `true` 的时候，停止运行;
- 主线程会在休眠一秒后将 `stop` 改为 `true`.

执行上面的代码发现, 即使 `stop` 已经变更为 `true`, `waiter` 线程也没有停止.

### 1.2 原因分析

![[双核 CPU 的系统架构简图.png|500]]

现代计算机一般是多核 CPU, 每个 CPU 都有自己的高速缓存, 其中主内存是共用的.

读取数据的时候, 如果未命中高速缓存, CPU 会从主内存中读取数据, 并刷到自己的高速缓存中; 操作共享变量的时候, 也是处理缓存中的数据, 处理完成后将其刷回主内存.

那这样, 我们就能找到上面问题发生的原因了: 主线程对变量的修改, 没有立刻写入 `waiter` 线程的缓存, 此时 `waiter` 依旧认为 `stop=false`.

## 2 JMM 模型

### 2.1 为什么需要 JMM?

上面提到的 CPU 多级缓存模型其实是一种规范, 基于这个规范会有 **多种实现**, 它们底层运行的指令集, 架构等会有所不同; 

Java 为了帮助开发者屏蔽掉差异, 在多级缓存模型之上, 设计了 Java 内存模型, 屏蔽底层变更, 这样就能实现 "一次编写, 到处运行".

![[直接内存与主内存.png|600]]

JMM 定义了一个规范, 每个线程都有自己的工作内存(直接内存), 线程操作共享变量时需要将其从主内存读取到工作内存, 操作完成后将变量刷新回主内存.

### 2.2 八种原子操作

CPU 操作内存是通过指令实现的, JMM 屏蔽掉这部分差异, 就也需要定义一组自己的指令:

| 操作          | 作用                           | 数据来源             | 数据去向     |
| ----------- | ---------------------------- | ---------------- | -------- |
| lock (锁定)   | 将主内存中的共享变量标记为某一线程独占的锁定状态     | —                | 主内存共享变量  |
| unlock (解锁) | 将处于锁定状态的共享变量解除锁定，使其他线程可再次访问  | —                | 主内存共享变量  |
| read (读取)   | 把主内存中的共享变量值传输到工作内存           | 主内存共享变量          | 工作内存     |
| load (载入)   | 把刚刚传输到工作内存的值赋给工作内存中的变量副本     | 工作内存(read 得到的数据) | 工作内存变量副本 |
| use (使用)    | 把工作内存变量副本的值提供给 CPU 执行引擎运算    | 工作内存变量副本         | CPU 执行引擎 |
| assign (赋值) | 把 CPU 执行引擎计算后的结果重新写回工作内存变量副本 | CPU 执行引擎         | 工作内存变量副本 |
| store (存储)  | 把工作内存中修改后的变量值传回主内存           | 工作内存变量副本         | 主内存      |
| write (写入)  | 将传回主内存的值正式写回到主内存的共享变量        | 主内存(store 得到的数据) | 主内存共享变量  |

### 2 volatile、synchronized


**有序性问题**：不同线程对共享变量的读写操作如何按合理的顺序进行，避免指令重排序带来的问题。

**原子性问题**：确保多个线程之间某些操作不可分割，避免出现竞争条件（race condition）。
我们继续讲完剩下的两个多线程问题，也就是有序性问题和原子性问题。
对于这两个问题，可以举一个单例模式双重检查锁的案例，双重检查锁的存在就是为了解决有序性问题和原子性问题：
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
创建对象的步骤其实是一个非原子性操作，通常包括以下几步：
- 分配内存空间；
- 调用构造方法，初始化对象；
- 将对象的引用指向分配的内存空间（将这个对象的引用赋值给 instance 变量）。
#### 2.1 synchronized 关键字
synchronized 保证了创建整个对象的操作是不可分割的，它确保在同一时刻，只有一个线程可以进入临界区（即创建实例的代码块），防止多个线程同时创建实例。
#### 2.2 volatile 关键字
但是由于处理器和编译器有可能会对上面的三条指令进行重排序，比如将第二步和第三步颠倒过来，这会导致在并发竞态环境下，可能会出现某些线程拿到的是一个不完整的对象。
volatile 关键字可以保证被修饰的变量不会被指令重排序影响；
具体来说，JVM 会在被 volatile 修饰的变量的写操作之前插入一个写屏障，在该变量的读操作之后插入一个读屏障。
- ==Load Barrier  读屏障==：在执行读操作之前，强制处理器刷新缓存，从主内存中读取最新的数据。
- ==Store Barrier 写屏障==：通过插入特定的指令，阻止处理器和编译器对写操作进行重排序，并且强制将写操作的结果从处理器的本地缓存刷新到主内存中。这样，其他线程在读取这些共享变量时，就能够读取到最新的值。
#### 2.3 总结
这两个关键字就是 JMM 的第二层抽象，我们都知道解决上面的问题需要保证 **原子性**、**可见性** 和 **顺序性**，那如何保证呢？
JMM 就为我们提供了 volatile、synchronized 等关键字，这些关键字是 JMM 规则在 Java 语言中的具体表现，它们提供了线程安全的基本工具，开发者不用在意系统底层的细节，只需要通过这些关键字，就能在上面 **直接内存** 和 **主内存** 结构模型的基础上，完成多线程程序的设计和编写。
### 3 Happens-Before 原则
JMM 可以通过 happens-before 关系向程序员提供跨线程的内存可见性保证，具体来说就是，如果 A 线程的写操作 a 与 B 线程的读操作 b 之间存在 happens-before 关系，尽管 a 操作和 b 操作在不同的线程中执行，但 JMM 向程序员保证 a 操作将对 b 操作可见。
具体的定义为：
- 如果一个操作 happens-before 另一个操作，那么第一个操作的执行结果将对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前。
- 两个操作之间存在 happens-before 关系，并不意味着 Java平 台的具体实现必须要按照 happens-before 关系指定的顺序来执行。如果重排序之后的执行结果，与按 happens-before 关系来执行的结果一致，那么这种重排序并不非法（也就是说，JMM允许这种重排序）。
具体的规则：
1. 程序顺序规则：一个线程中的每个操作，happens-before 于该线程中的任意后续操作。
2. 监视器锁规则：对一个锁的解锁，happens-before 于随后对这个锁的加锁。
3. volatile 变量规则：对一个 volatile 域的写，happens-before 于任意后续对这个volatile域的读。
4. 传递性：如果 A happens-before B，且 B happens-before C，那么 A happens-before C。
5. start 规则：如果线程A执行操作 ThreadB.start（启动线程B），那么A线程的 ThreadB.start 操作 happens-before 于线程B中的任意操作。
6. Join 规则：如果线程A执行操作 ThreadB.join 并成功返回，那么线程B中的任意操作 happens-before 于线程A从 ThreadB.join() 操作成功返回。
7. 程序中断规则：对线程interrupted()方法的调用先行于被中断线程的代码检测到中断时间的发生。
8. 对象 finalize 规则：一个对象的初始化完成（构造函数执行结束）先行于发生它的 finalize 方法的开始。
### 4 synchronized 的实现原理
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

