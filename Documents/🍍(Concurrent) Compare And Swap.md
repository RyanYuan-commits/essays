---
type: ConcurrentProgramming
sub-type: Basic
---
[[🥒(Java) AbstractQueueSynchronizer]]
## 1 背景

### 1.1 解决锁性能问题

Java 为保证变量原子性, 提供了如 `synchronized`、`Lock` 等锁机制. 

其原理是阻塞线程, 当一个线程持有锁执行任务时, 其他线程会被阻塞等待. 在高并发场景下, 大量线程频繁等待锁、获取锁、释放锁, 会导致严重的上下文切换开销, 造成性能下降. 

比如在秒杀系统中, 大量用户请求同一资源, 如果用传统锁机制, 系统性能会大幅降低. 而 CAS 是一种无锁算法, 它通过硬件级别的原子操作实现多线程同步, 避免了线程阻塞, 大大提升了并发性能. 

### 1.2 实现乐观锁机制

乐观锁假设在大多数情况下不会有冲突, 每次不加锁而是直接尝试完成某项操作, 如果失败就重试, 直到成功. 

CAS 是乐观锁的一种实现方式, 它在更新数据时, 先比较内存中的值与预期值, 如果相等才更新为新值, 否则不操作. 

在一些**读多写少**的场景(如缓存数据更新)中, 使用基于 CAS 的乐观锁机制, 能减少锁竞争, 提升系统并发处理能力. 

## 2 CAS in Java

CAS 全称是 Compare And Swap, 是一种用于**在多线程环境下实现同步功能的机制**. 

CAS 操作包含三个操作数, 分别是内存位置、预期数值和新值. CAS 的实现逻辑是将内存位置处的数值与预期数值想比较, 若相等, 则将内存位置处的值替换为新值. 若不相等, 则不做任何操作. 

Java 语言库的 CAS 是通过 C++ 内联汇编的形式实现的. [[🍦(Java) Unsafe|Unsafe]] 提供的 CAS 方法底层为 CPU 指令 `cmpxchg`.

Java 中的大部分原子类都是通过 CAS 来实现的: 

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final Unsafe unsafe = Unsafe.getUnsafe();    
    private static final long valueOffset;   
    static {       
        try {            
            // 计算变量 value 在类对象中的偏移
            valueOffset = unsafe.objectFieldOffset(AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { 
	        throw new Error(ex); 
        }
    }        
    private volatile int value; 

	
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }    
}
```

## 3 CAS 存在的问题

### 3.1 ABA 问题

谈到 CAS, 基本上都要谈一下 CAS 的 ABA 问题, 简单来说就是变量被从 A 改成 B 然后又改回 A 的时候, 线程无法感知到这个过程. 
ABA问题的解决思路其实也很简单, 就是使用版本号. 在变量前面追加上版本号, 每次变量更新的时候把版本号加 1, A→B→A 就会变成 1A→2B→3A 了. 

从 Java1.5 开始 atomic 包里提供了一个类 `AtomicStampedReference` 来解决 ABA 问题. 这个类的 `compareAndSet` 方法作用是首先检查当前引用是否等于预期引用, 并且当前标志是否等于预期标志, 如果全部相等, 则以原子方式将该引用和该标志的值设置为给定的更新值. 

### 3.2 循环时间长开销大

自旋 CAS如果长时间不成功, 会给 CPU 带来非常大的执行开销. 如果 JVM 能支持处理器提供的 `pause` 指令那么效率会有一定的提升;

`pause` 指令有两个作用:

可以延迟流水线执行指令, 使 CPU 不会消耗过多的执行资源, 延迟的时间取决于具体实现的版本, 在一些处理器上延迟时间是零. 

可以避免在退出循环的时候因内存顺序冲突而引起 CPU 流水线被清空, 从而提高 CPU 的执行效率. 

### 3.3 只能保证一个共享变量的原子操作

当对一个共享变量执行操作时, 我们可以使用循环 CAS 的方式来保证原子操作, 但是对多个共享变量操作时, CAS 无法保证操作的原子性, 此时, 可以把多个共享变量合并成一个共享变量来操作, 比如有两个共享变量 `i＝2; j=a`, 可以合并为 `ij=2a`, 然后用 CAS 来操作 `ij`. 

从 Java1.5 开始 JDK 提供了 `AtomicReference` 类来保证引用对象之间的原子性, 你可以把多个变量放在一个对象里来进行 CAS 操作. 