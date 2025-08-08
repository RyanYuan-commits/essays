## 1 堆内存溢出
Java堆内存的 `OutOfMemoryError` 异常是实际应用中最常见的内存溢出异常情况。
出现 Java 堆内存溢出时，异常堆栈信息 `java.lang.OutOfMemoryError` 会跟随进一步提示 `Java heap space`。
```java
/**
 * @author kq
 * 2024-08-16 14:58
 * VM Args: -Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
 **/
@SuppressWarnings("all")
public class HeapOOM {
    static class OOMObject {
        // 1MB
        public byte[] bytes = new byte[1024*1024];
    }
    public static void main(String[] args) {
        ArrayList<OOMObject> oomObjects = new ArrayList<>();
        while(true){
            oomObjects.add(new OOMObject());
        }
    }
}
```
运行结果：
```
java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid31577.hprof ...
Heap dump file created [22016937 bytes in 0.012 secs]
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at com.ryan.HeapOOM$OOMObject.<init>(HeapOOM.java:15)
	at com.ryan.HeapOOM.main(HeapOOM.java:22)
```
要解决这个内存区域的异常，常规的处理方法是首先通过内存映像分析工具对 Dump 出来的堆转储快照进行分析。第一步首先应确认内存中导致OOM 的对象是否是必要的，也就是要先分清楚到底是出现了内存泄漏（Memory Leak）还是内存溢出（MemoryOverflow）。
- **如果是内存泄漏**（是指程序中已分配的内存空间未能得到释放，导致这部分内存无法被再次利用）可进一步通过工具查看泄漏对象到GC Roots的引用链，找到泄漏对象是通过怎样的引用路径、与哪些GC Roots相关联，才导致垃圾收集器无法回收它们，根据泄漏对象的类型信息以及它到 GC Roots 引用链的信息，一般可以比较准确地定位到这些对象创建的位置，进而找出产生内存泄漏的代码的具体位置。
- **如果不是内存泄漏**，换句话说就是内存中的对象确实都是必须存活的，那就应当检查Java虚拟机的堆参数（-Xmx与-Xms）设置，与机器的内存对比，看看是否还有向上调整的空间。再从代码上检查是否存在某些对象生命周期过长、持有状态时间过长、存储结构设计不合理等情况，尽量减少程序运行期的内存消耗
## 虚拟机栈和本地方法栈溢出
```java
/**
 * @author kq
 * 2024-08-16 15:40
 * 设置线程栈的大小 VM Args: -Xss640k
 **/
@SuppressWarnings("all")
public class JavaVMStackSOF {
    private int stackLength = 1;
    public void stackLeak(){
        stackLength++;
        stackLeak();
    }
    public static void main(String[] args) {
        JavaVMStackSOF oom = new JavaVMStackSOF();
        try {
            oom.stackLeak();
        }catch (Throwable e){
            System.out.println("stack length:"+oom.stackLength);
            throw e;
        }
    }

}
```
运行结果
```
stack length:4924
Exception in thread "main" java.lang.StackOverflowError
	at com.ryan.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:15)
	at com.ryan.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:15)
	at com.ryan.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:15)
	at com.ryan.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:15)
	at com.ryan.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:15)
	// 。。。 其他栈信息
```
如果线程请求的栈深度大于虚拟机所允许的最大深度，无论是由于栈帧太大还是虚拟机栈容量太小，当新的栈帧内存无法分配的时候，HotSpot 虚拟机抛出的都是 `StackOverflowError` 异常。而如果虚拟机的栈内存允许动态扩展，当扩展栈容量无法申请到足够的内存时，将抛出`OutOfMemoryError` 异常。
虽然 HotSpot 虚拟机不支持线程的动态拓展，但是如果创建的线程过多，同样会引发 `OutOfMemoryError` 异常，原因其实不难理解，操作系统分配给每个进程的内存是有限制的，譬如32位Windows的单个进程最大内存限制为2GB。HotSpot 虚拟机提供了参数可以控制 Java 堆和方法区这两部分的内存的最大值，那剩余的内存即为 2GB（操作系统限制）减去最大堆容量，再减去最大方法区容量，由于程序计数器消耗内存很小，可以忽略掉，如果把直接内存和虚拟机进程本身耗费的内存也去掉的话，剩下的内存就由虚拟机栈和本地方法栈来分配了。
因此为每个线程分配到的栈内存越大，可以建立的线程数量自然就越少，建立线程时就越容易把剩下的内存耗尽。
```java
/**
 * @author kq
 * 2024-08-16 15:40
 * VM Args: -Xss2m
 **/
@SuppressWarnings("all")
public class JavaVMStackOOM {
    private void donotStop() {
        while (true) {}
    }
    public void stackLeakByThread() {
        while (true) {
            Thread thread = new Thread(new Runnable() {
                @Override
                public void run() {
                    donotStop();
                }
            });
            thread.start();
        }
    }
    public static void main(String[] args) {
        JavaVMStackOOM oom = new JavaVMStackOOM();
        oom.stackLeakByThread();
    }
}
```
运行结果
```
Exception in thread "main" java.lang.OutOfMemoryError: unable to create new native thread
	at java.lang.Thread.start0(Native Method)
	at java.lang.Thread.start(Thread.java:719)
	at com.ryan.JavaVMStackOOM.stackLeakByThread(JavaVMStackOOM.java:23)
	at com.ryan.JavaVMStackOOM.main(JavaVMStackOOM.java:29)
```
# 方法区和运行时常量池溢出
由于运行时常量池是方法区的一部分，所以这两个区域的溢出测试可以放到一起进行。前面曾经提到HotSpot从JDK 7开始逐步“去永久代”的计划，并在JDK 8中完全使用元空间来代替永久代的背景故事，在此我们就以测试代码来观察一下，使用“永久代”还是“元空间”来实现方法区，对程序有什么实际的影响。
![[方法区存储的内容.png|400]]

JDK7 及之前，将方法区存放在堆区域的**永久代空间**，堆的大小由虚拟机参数来控制
JDK8 及其之后的版本，将方法区存储在**元空间**中，元空间位于操作系统维护的直接内存中，默认情况下只要不超过操作系统承受的上限，可以一直分配。
## 运行时常量池导致的内存溢出异常
`String::intern()`是一个本地方法，它的作用是如果字符串常量池中已经包含一个等于此String对象的字符串，则返回代表池中这个字符串的String对象的引用；否则，会将此String对象包含的字符串添加到常量池中，并且返回此String对象的引用。在JDK 6或更早之前的HotSpot虚拟机中，常量池都是分配在永久代中，我们可以通过 `-XX:PermSize` 和 `-XX:MaxPermSize` 限制永久代的大小，即可间接限制其中常量池的容量，
```java
/*
 * VM Args：-XX:PermSize=6M -XX:MaxPermSize=6M
 * @author zzm
 */
public class RuntimeConstantPoolOOM {
	 public static void main(String[] args) {
	 // 使用Set保持着常量池引用，避免Full GC回收常量池行为
	 Set<String> set = new HashSet<>();
	 // 在short范围内足以让6MB的PermSize产生OOM了
		short i = 0;
		 while (true) {
			 set.add(String.valueOf(i++).intern());
		 }
	 }
}
```
运行结果
```
Exception in thread "main" java.lang.OutOfMemoryError: PermGen space
 at java.lang.String.intern(Native Method)
 at org.fenixsoft.oom.RuntimeConstantPoolOOM.main(RuntimeConstantPoolOOM.java: 18)
```
> 运行代码及结果出自《深入理解 Java 虚拟机》原书第三版。

而使用JDK 7或更高版本的JDK来运行这段程序并不会得到相同的结果，无论是在JDK 7中继续使用 `-XX:MaxPermSize` 参数或者在JDK 8及以上版本使用 `-XX:MaxMeta-spaceSize` 参数把方法区容量同样限制在 6MB，也都不会重现JDK 6中的溢出异常，循环将一直进行下去，永不停歇。
出现这种变化，是因为自JDK 7起，**原本存放在永久代的字符串常量池被移至Java堆之中**，所以在JDK 7及以上版本，限制方法区的容量对该测试用例来说是毫无意义的。
这时候使用-Xmx参数限制最大堆到6MB就能够看到以下两种运行结果之一，具体取决于哪里的对象分配时产生了溢出：
```
// OOM异常一：
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
 at java.base/java.lang.Integer.toString(Integer.java:440)
 at java.base/java.lang.String.valueOf(String.java:3058)
 at RuntimeConstantPoolOOM.main(RuntimeConstantPoolOOM.java:12)
 
// OOM异常二：
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
 at java.base/java.util.HashMap.resize(HashMap.java:699)
 at java.base/java.util.HashMap.putVal(HashMap.java:658)
 at java.base/java.util.HashMap.put(HashMap.java:607)
 at java.base/java.util.HashSet.add(HashSet.java:220)
 at RuntimeConstantPoolOOM.main(RuntimeConstantPoolOOM.java from InputFile-Object:14)
```
关于这个字符串常量池的实现在哪里出现问题，还可以引申出一些更有意思的影响
```java
public class RuntimeConstantPoolOOM {
		public static void main(String[] args) {
			String str1 = new StringBuilder("计算机").append("软件").toString();
			System.out.println(str1.intern() == str1);
			String str2 = new StringBuilder("ja").append("va").toString();
			System.out.println(str2.intern() == str2);
		}
}
```
这段代码在JDK 6中运行，会得到两个false，而在JDK 7中运行，会得到一个true和一个false。
产生差异的原因是，在JDK 6中，**`intern()`**方法会把**首次遇到的字符串实例**复制到永久代的字符串常量池中存储，返回的也是永久代里面这个字符串实例的引用，而由**`StringBuilder`** 创建的字符串对象实例在Java堆上，所以必然不可能是同一个引用，结果将返回false。
而 JDK 7（以及部分其他虚拟机，例如JRockit）的 **`intern()`** 方法实现就不需要再拷贝字符串的实例到永久代了，既然字符串常量池已经移到 Java 堆中，那只需要在常量池里记录一下首次出现的实例引用即可，因此 **`intern()`** 返回的引用和由 **`StringBuilder`** 创建的那个字符串实例就是同一个。而对str2比较返回false，这是因为“java”[2]这个字符串在执行 **`StringBuilder.toString()`** 之前就已经出现过了，字符串常量池中已经有它的引用，不符合 **`intern()`** 方法要求“首次遇到”的原则，“计算机软件”这个字符串则是首次出现的，因此结果返回 true。
## 类信息导致的内存溢出异常
我们再来看看方法区的其他部分的内容，方法区的主要职责是用于存放类型的相关信息，如类名、访问修饰符、常量池、字段描述、方法描述等。对于这部分区域的测试，基本的思路是运行时产生大量的类去填满方法区，直到溢出为止。虽然直接使用Java SE API也可以动态产生类（如反射时的 `GeneratedConstructorAccessor` 和动态代理等），但在本次实验中操作起来比较麻烦。
```java
/**
 * VM Args：-XX:PermSize=10M -XX:MaxPermSize=10M
 * @author zzm
 */
public class JavaMethodAreaOOM {
    public static void main(String[] args) {
        while (true) {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(OOMObject.class);
            enhancer.setUseCache(false);
            enhancer.setCallback(new MethodInterceptor() {
                public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
                    return proxy.invokeSuper(obj, args);
                }
            });
            enhancer.create();
        }
    }
    static class OOMObject {}
}
```
在 JDK7 中的运行结果：
```
Caused by: java.lang.OutOfMemoryError: PermGen space
 at java.lang.ClassLoader.defineClass1(Native Method)
 at java.lang.ClassLoader.defineClassCond(ClassLoader.java:632)
 at java.lang.ClassLoader.defineClass(ClassLoader.java:616)
 ... 8 more
```
方法区溢出也是一种常见的内存溢出异常，一个类如果要被垃圾收集器回收，要达成的条件是比较苛刻的。
在经常运行时生成大量动态类的应用场景里，就应该特别关注这些类的回收状况。这类场景除了之前提到的程序使用了CGLib字节码增强和动态语言外，常见的还有：大量JSP或动态产生JSP文件的应用（JSP第一次运行时需要编译为Java类）、基于OSGi的应用（即使是同一个类文件，被不同的加载器加载也会视为不同的类）等。 
在JDK 8以后，永久代便完全退出了历史舞台，元空间作为其替代者登场。在默认设置下，前面列举的那些正常的动态创建新类型的测试用例已经很难再迫使虚拟机产生方法区的溢出异常了。不过为了让使用者有预防实际应用里出现类似于代码清单2-9那样的破坏性的操作，HotSpot还是提供了一些参数作为元空间的防御措施，主要包括：
- **`-XX:MaxMetaspaceSize`**：设置元空间最大值，默认是-1，即不限制，或者说只受限于本地内存大小。
- **`-XX：MetaspaceSize`**：指定元空间的初始空间大小，以字节为单位，达到该值就会触发垃圾收集进行类型卸载，同时收集器会对该值进行调整：如果释放了大量的空间，就适当降低该值；如果释放了很少的空间，那么在不超过 **`-XX:MaxMetaspaceSize`**（如果设置了的话）的情况下，适当提高该值。
- **`-XX：MinMetaspaceFreeRatio`**：作用设置了一个最小的空闲比例，单位是百分比，当元空间的空闲比例低于这个设定值时，JVM会尝试释放一些不再使用的元数据，以增加空闲空间，可减少因为元空间不足导致的垃圾收集的频率。类似的还有**`-XX:Max-MetaspaceFreeRatio`，用于控制最大的元空间剩余容量的百分比；将其设置的大一点可以避免多次的扩容，保证方法区始终有足够的空余内存空间。
# 本机直接内存溢出
直接内存（Direct Memory）的容量大小可通过 `-XX:MaxDirectMemorySize` 参数来指定，如果不去指定，则默认与Java堆最大值（由-Xmx指定）一致。
JVM 的直接内存（Direct Memory）是指Java虚拟机（JVM）中的一种特殊的内存区域，它不在Java堆（Heap）中分配，也不受垃圾回收器（Garbage Collector）管理。直接内存主要用于实现高性能的I/O操作，尤其是在处理大量数据传输时，如网络通信和文件 I/O。
代码越过了 `DirectByteBuffer` 类直接通过反射获取 `Unsafe` 实例进行内存分配，因为虽然使用 `DirectByteBuffer` 分配内存也会抛出内存溢出异常，但它抛出异常时并没有真正向操作系统申请分配内存，而是通过计算得知内存无法分配就会在代码里手动抛出溢出异常，真正申请分配内存的方法是 `Unsafe::allocateMemory()`。
```java
/* 
 * VM Args -XX:MaxDirectMemorySize=11M
 * @author zzm
 */
 public class DirectMemoryOOM {  
    private static final int _1MB = 1024 * 1024;  
    public static void main(String[] args) {  
        ByteBuffer byteBuffer = ByteBuffer.allocateDirect(11 * _1MB);  
    }  
}
```

>[!info] DirectByteBuffer.DirectByteBuffer()
>当执行 allocate 方法后，执行的 DirectByteBuffer 的构造方法。
```java
DirectByteBuffer(int cap) {                   // package-private  
    super(-1, 0, cap, cap);  
    boolean pa = VM.isDirectMemoryPageAligned();  
    int ps = Bits.pageSize();  
    long size = Math.max(1L, (long)cap + (pa ? ps : 0));  
    Bits.reserveMemory(size, cap);  
    long base = 0;  
    try {  
        base = unsafe.allocateMemory(size);  
    } catch (OutOfMemoryError x) {  
        Bits.unreserveMemory(size, cap);  
        throw x;  
    }  
    unsafe.setMemory(base, size, (byte) 0);  
    if (pa && (base % ps != 0)) {  
        // Round up to page boundary  
        address = base + ps - (base & (ps - 1));  
    } else {  
        address = base;  
    }  
    cleaner = Cleaner.create(this, new Deallocator(base, size, cap));  
    att = null;  
}
```
由直接内存导致的内存溢出，一个明显的特征是在Heap Dump文件中不会看见有什么明显的异常情况，如果读者发现内存溢出之后产生的Dump文件很小，而程序中又直接或间接使用了 **`DirectMemory`**（典型的间接使用就是NIO），那就可以考虑重点检查一下直接内存方面的原因了。