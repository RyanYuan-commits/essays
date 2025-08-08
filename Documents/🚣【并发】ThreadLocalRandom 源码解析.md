## Random 类的局限性
在 JDK1.7 之前，java.util.Random 都是应用比较广泛的随机数生成工具，java.lang.Math 中的随机数生成也是使用的 Random 的实例。
写一段代码来测试一下 Random 实例：
```java
public class RandomTest {
    public static void main(String[] args) {
        Random random = new Random();
        for (int i = 1; i < 10; i++) {
            // 输出 10 个在 0 到 5 之间的随机数，不包括 5
            System.out.println(random.nextInt(5));
        }
    }
}
```

Random 类生成随机数主要依靠 next 方法：
```java
protected int next(int bits) {
    long oldseed, nextseed;
    AtomicLong seed = this.seed;
    do {
        oldseed = seed.get();
        nextseed = (oldseed * multiplier + addend) & mask;
    } while (!seed.compareAndSet(oldseed, nextseed));
    return (int)(nextseed >>> (48 - bits));
}
```
这个方法用于生成一个指定位数的随机数，随机数的生成依赖于 seed 种子的生成，nextseed = (oldseed * multiplier + addend) & mask;。
这里使用了线性同余算法来生成伪随机数，即将当前种子值乘以一个常数 multiplier，然后加上另一个常数 addend，最后对结果进行位与操作并截取低位以得到下一个种子值。
在多线程环境下，java.util.Random 类可能会生成相同的随机数的原因主要是由于 共享同一个种子 和 竞争条件 导致的；所以 ThreadLocalRandom 应运而生。
## ThreadLocalRandom
先写一段代码来展示如何使用它：
```java
public class ThreadLocalRandomTest {
    public static void main(String[] args) {
        ThreadLocalRandom random = ThreadLocalRandom.current();
        for (int i = 0; i < 10; i++) {
        // 生成随机数
            System.out.println(random.nextInt(5));
        }
    }
}
```
看到这个名字，很容易就能联想到 ThreadLocal，ThreadLocal 的原理是让每个线程复制一份变量，每个线程操作自己的副本，从而避免多个线程之间的同步问题。
ThreadLocalRandom 同样也是用了这种方法，Random 的缺点就是多个线程使用一个 seed，从而引发竞争的情况；ThreadLocalRandom 在每个线程中去维护一个种子，从而避免了这个问题；
	ThreadLocalRandom 继承了 Random 类并且重写了 `nextInt()` 方法
	ThreadLocalRandom 类中使用的种子存放在调用线程的 threadLocalRandomSeed 变量中（类比 ThreadLocal）；
当线程调用了 ThreadLocalRandom 类的 current() 方法的时候，ThreadLocalRandom 会去初始化调用线程的 threadLocalRandomSeed 变量，也就是初始化种子。
## nextSeed() 方法
nextInt 方法和 Random 实现基本相同，重点来关注一下 `nextSeed()` 方法。
```java
final long nextSeed() {
	Thread t; long r; // read and update per-thread seed
	UNSAFE.putLong(t = Thread.currentThread(), SEED,
				   r = UNSAFE.getLong(t, SEED) + GAMMA);
	return r;
}
```
拆分开来看，这个方法就是将线程中偏移量为 SEED 的属性的值变为了原本的种子值加上 GAMMA，然后将这个新的种子值返回，GAMMA 的注释为 The seed increment，也就是 seed 的增量。