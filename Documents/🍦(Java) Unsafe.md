---
type: Java
sub-type: JavaSE
finished: "false"
created: 2025-09-28 22:54:09
updated: 2025-10-02 22:59:04
---
`Unsafe` 是位于 `sun.misc` 包下的类, 主要提供一些用于执行低级别, 不安全操作的方法, 如直接访问系统内存资源, 自主管理内存资源等.

## 1	基本介绍

```java
public final class Unsafe {

	private static final Unsafe theUnsafe = new Unsafe();
	
	@CallerSensitive  
	public static Unsafe getUnsafe() {  
	    Class<?> caller = Reflection.getCallerClass();  
	    if (!VM.isSystemDomainLoader(caller.getClassLoader()))  
	        throw new SecurityException("Unsafe");  
	    return theUnsafe;  
	}
}
```

`Unsafe` 类为单例实现, 提供静态方法 `getUnsafe` 获取 `Unsafe` 实例;

但是在我们的应用程序中, 是无法直接获取到 `Unsafe` 实例的, 因为它会检查方法调用者是否是 bootstrap 或者 platform 类加载器加载的;

获取实例有两种可行方案:

其一, 将调用 `Unsafe` 的类追加到默认的 bootstrap 路径中: 

```
java -Xbootclasspath/a: ${path} // 其中path为调用Unsafe相关方法的类所在jar包路径
```

其二, 通过反射获取单例对象 `theUnsafe`, 在 JDK9 之后, `sun.misc` 被放置在 `jdk.unsupported` 中并开放反射:

```java
// ====== jdk.unsupported module.info 文件 ======
module jdk.unsupported {  
    exports com.sun.nio.file;  
    exports sun.misc;  
    exports sun.reflect;  
  
	// 对外开放反射
    opens sun.misc;  
    opens sun.reflect;  
}

// ====== 通过反射获取 Unsafe 类 ======
Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");  
theUnsafe.setAccessible(true);  
Unsafe unsafe = (Unsafe) theUnsafe.get(null);  
System.out.println(unsafe); // sun.misc.Unsafe@7e9e5f8a
```

## 2	功能介绍

![[Unsafe 功能介绍.png|900]]

## 3	内存操作

这部分主要包含对堆外内存的分配, 拷贝, 释放, 给定地址值操作等方法.

```java
// 分配内存, 相当于 C++ 的 malloc 函数
public native long allocateMemory(long bytes);

// 扩充内存
public native long reallocateMemory(long address, long bytes);

// 释放内存
public native void freeMemory(long address);

// 在给定的内存块中设置值
public native void setMemory(Object o, long offset, long bytes, byte value);

// 内存拷贝
public native void copyMemory(Object srcBase, long srcOffset, Object destBase, long destOffset, long bytes);

// 获取给定地址值，忽略修饰限定符的访问限制。与此类似操作还有: getInt，getDouble，getLong，getChar等
public native Object getObject(Object o, long offset);

// 为给定地址设置值，忽略修饰限定符的访问限制，与此类似操作还有: putInt,putDouble，putLong，putChar等
public native void putObject(Object o, long offset, Object x);

// 获取给定地址的byte类型的值（当且仅当该内存地址为allocateMemory分配时，此方法结果为确定的）
public native byte getByte(long address);

// 为给定地址设置byte类型的值（当且仅当该内存地址为allocateMemory分配时，此方法结果才是确定的）
public native void putByte(long address, byte x);
```

Java 对堆外内存这种存在于 JVM 管控之外的内存区域的操作, 依赖于 `Unsafe` 提供的 `native` 方法;

在 I/O 过程中, 会存在堆内外的拷贝操作, 对于频繁进行内存间数据拷贝, 且生命周期较短的暂存数据, 都建议存储到堆外内存;

典型应用: [[🎙️(Java) DirectByteBuffer|DirectByteBuffer]]

## 4	[[🍍(Concurrent) Compare And Swap|CAS]] 相关

```java
public final native boolean compareAndSwapObject(Object o, long offset,  Object expected, Object update);

public final native boolean compareAndSwapInt(Object o, long offset, int expected,int update);
  
public final native boolean compareAndSwapLong(Object o, long offset, long expected, long update);
```

CAS 在 `java.util.concurrent.atomic` 相关类、`Java AQS`、`ConcurrentHashMap` 等实现上有非常广泛的应用

![[🍍(Concurrent) Compare And Swap#2 CAS in Java|CAS in Java]]
## 5	线程调度

```java
// 取消阻塞线程
public native void unpark(Object thread);

// 阻塞线程
public native void park(boolean isAbsolute, long time);

// 获得对象锁 (可重入锁)
@Deprecated
public native void monitorEnter(Object o);

// 释放对象锁
@Deprecated
public native void monitorExit(Object o);

// 尝试获取对象锁
@Deprecated
public native boolean tryMonitorEnter(Object o);
```

方法 `park` 和 `unpark` 即可实现线程的挂起和恢复, 将一个线程进行挂起是通过 `park` 方法实现的.

## 6	Class 相关

### 6.1	概述

这部分主要提供 Class 和它的静态字段的操作相关方法, 包括静态字段内存定位, 定义类, 定义匿名类, 检验 & 确保初始化等.

```java
//获取给定静态字段的内存地址偏移量，这个值对于给定的字段是唯一且固定不变的
public native long staticFieldOffset(Field f);
//获取一个静态类中给定字段的对象指针
public native Object staticFieldBase(Field f);
//判断是否需要初始化一个类，通常在获取一个类的静态属性的时候（因为一个类如果没初始化，它的静态属性也不会初始化）使用。 当且仅当ensureClassInitialized方法不生效时返回false。
public native boolean shouldBeInitialized(Class<?> c);
//检测给定的类是否已经初始化。通常在获取一个类的静态属性的时候（因为一个类如果没初始化，它的静态属性也不会初始化）使用。
public native void ensureClassInitialized(Class<?> c);
//定义一个类，此方法会跳过JVM的所有安全检查，默认情况下，ClassLoader（类加载器）和ProtectionDomain（保护域）实例来源于调用者
public native Class<?> defineClass(String name, byte[] b, int off, int len, ClassLoader loader, ProtectionDomain protectionDomain);
//定义一个匿名类
public native Class<?> defineAnonymousClass(Class<?> hostClass, byte[] data, Object[] cpPatches);
```

### 6.2	典型应用

从 JDK8 开始, JDK 使用 Invokedynamic 以及 VM Anonymous Class 结合来实现 Java 语言层面的 Lambda 表达式.

- **Invokedynamic**: Java7 为了是现在 JVM 上运行动态语言而引入的一条新的虚拟机指令, 它可以实现在运行期动态解析出调用点限定符所引用的方法, 然后再执行该方法, Invokedynamic 指令的分派逻辑是由用户设定的引导方法决定的.
- **VM Anonymous Class**: 一种模板机制, 当程序动态生成很多结构相同, 只有若干常量不同的类时, 可以先创建包含常量占位符的模板类, 而后通过 Unsafe.defineAnonymousClass 方法定义具体类时填充模板占位符来生成具体的匿名类; 生成的匿名类不显示的挂在任何的 ClassLoader 下, 只要该类没有存在的实例, 且没有强引用来引用该类的 Class 对象, 这个类就会被 GC 回收.

 在 Lambda 表达式中, 通过 invokedynamic 指令调用引导方法生成调用点, 在此过程中, 会通过 ASM 动态生成字节码, 然后通过 Unsafe 的 defineAnonymousClass 方法定义实现相应的函数式接口的匿名类, 然后再实例化该匿名类, 返回与此匿名类中函数式方法的方法句柄关联的调用点, 而后可以通过此调用点实现调用相应 Lambda 表达式定义逻辑的功能:

```java
public class UnsafeDemo {  
  
    public static void main(String[] args) {  
        Consumer<String> consumer = s -> System.out.println(s);  
        consumer.accept("hello world");  
    }  
  
}

// ====== 反编译 ======

Compiled from "UnsafeDemo.java"
public class com.ryan.demo.UnsafeDemo {
  public com.ryan.demo.UnsafeDemo();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: invokedynamic #7,  0              // InvokeDynamic #0:accept:()Ljava/util/function/Consumer;
       5: astore_1
       6: aload_1
       7: ldc           #11                 // String hello world
       9: invokeinterface #13,  2           // InterfaceMethod java/util/function/Consumer.accept:(Ljava/lang/Object;)V
      14: return
}
```

![[Pasted image 20251013210656.png]]