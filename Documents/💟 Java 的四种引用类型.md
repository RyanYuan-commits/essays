![[Java 中的四种引用类型.png|900]]
# 强引用（Strong Reference）
创建一个对象并把这个对象赋给一个引用变量（我们用的最多的等号+new关键字），如果内存不足，**JVM会抛出OOM错误也不会回收对象**， 想中断强引用和某个对象之间的关联，可以显示地将引用赋值为null。
```java
/**  
 * @author Ryan Yuan 
 * 2024-10-17 14:09 
 * -Xmx5m -Xms5m 
 **/
 public class Main {  
    public static void main(String[] args) {  
        A a = new A();  
        try {  
	            byte[] bytes = new byte[1024 * 1024 * 8];  
        } catch (Throwable t) {  
            t.printStackTrace();  
        } finally {  
            System.out.println(a);  
        }  
    }  
    static class A {}  
}
```
在上面的案例中，我们将堆中的启始内存和最大内存都设置为 50MB，然后构建一个强引用的对象；然后创建一个 8MB 的数组到堆中（内存溢出），此时使用 finally 来查看 a 是什么：
可以看到即使内存溢出，触发的 GC 也不会将 a 引用的对象清除。
# 软引用（Soft Reference）
如果一个对象具有软引用，内存空间足够，垃圾回收器就不会回收它；**如果内存空间不足了，就会回收这些对象的内存**。
软引用可用来实现内存敏感的高速缓存,比如网页缓存、图片缓存等，使用软引用能防止内存泄露，增强程序的健壮性。
Java 中可以通过 `SoftReference` 类来实现软引用，同样来看一个例子：
```java
/**
 * @author Ryan Yuan
 * 2024-10-17 14:09
 * -Xmx5m -Xms5m
 **/
public class Main {  
    public static void main(String[] args) {  
        SoftReference<A> a = new SoftReference<>(new A());  
        try {  
            byte[] bytes = new byte[1024 * 1024 * 8];  
        } catch (Throwable t) {  
            t.printStackTrace();  
        } finally {  
            System.out.println(a.get());  
        }  
    }  
    static class A {}  
}
```
输出的结果是这样：
```
java.lang.OutOfMemoryError: Java heap space
	at com.kq.primitive.main(primitive.java:15)
null
```
此时当内存溢出的时候，软引用的对象就会被清除了。
# 弱引用（Weak Reference）
当JVM进行垃圾回收时，无论内存是否充足，都会回收被弱引用关联的对象
弱引用通常是用来描述非必需对象。
```java
public class Main {  
    public static void main(String[] args) throws IOException {  
        WeakReference<A> a = new WeakReference<>(new A());  
        try {  
            System.gc();  
        } catch (Throwable t) {  
            t.printStackTrace();  
        } finally {  
            System.out.println(a.get());  
        }  
    }  
    static class A {}  
}
```
这里我们手动调用 `System.gc()` 来触发 GC，观察一下弱引用的效果：
```
null
```
可以看到直接输出了 `null` 。
# 虚引用（Phantom Reference）
与其他引用类型不同，`PhantomReference` 的 `get()` 方法总是返回 `null`。因此，无法通过 `PhantomReference` 访问其引用的对象。
`PhantomReference` 的主要用途是跟踪对象被垃圾回收的过程。它通常与 `ReferenceQueue` 一起使用。当垃圾回收器准备回收一个对象时，如果发现它有一个 `PhantomReference`，就会在回收对象之前，将这个引用加入到与之关联的 `ReferenceQueue` 中。
```java
/**
 * @author Ryan Yuan
 * 2024-10-17 14:09
 **/
public class Main {
    public static void main(String[] args) throws InterruptedException {
        // Strong Reference
        A a = new A();
        
        ReferenceQueue<A> stuReferenceQueue = new ReferenceQueue<>();
        PhantomReference<A> stuPhantomReference = new PhantomReference<>(a, stuReferenceQueue);
        // You can not get object from PhantomReference
        System.out.println(stuPhantomReference.get());
        // PhantomReference.isEnqueued()：
        // Tells whether this reference object has been enqueued, either by the program or by the garbage collector.
        System.out.println(stuPhantomReference.isEnqueued());

        // Help gc
        a = null;
        System.gc();
        Thread.sleep(50);
        System.out.println(stuPhantomReference.isEnqueued());
    }
    static class A{}
}
```