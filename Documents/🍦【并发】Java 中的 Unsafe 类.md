## Unsafe 类中的重要方法
rt.jar 中的 Unsafe 类提供了硬件级别的原子行操作，其中的方法都是 native 方法，使用了 JNI 的方式访问本地的 C++ 实现库。
JNI（Java Native Interface）是 Java 提供的一种编程框架，用于在 Java 虚拟机（JVM）中调用本地（即非 Java）代码。它允许 Java 程序与其他语言（如C、C++）编写的代码进行交互。
通过 JNI，Java 程序可以调用本地方法来执行一些特定的操作，例如访问系统资源、调用操作系统的特定功能、使用硬件设备等。同时，本地代码也可以调用 Java 方法，这样就可以在 Java 环境中利用其他语言的强大功能。

JNI 的基本工作流程包括以下几个步骤：
1. 编写本地代码：使用 C、C++ 或其他支持的编程语言编写需要被调用的本地方法的实现。
2. 编写 JNI 头文件：为了让 Java 程序知道如何调用本地方法，需要编写一个对应的 JNI 头文件（*.h 文件），其中声明了本地方法的函数原型。
3. 实现本地方法：在本地代码中实现 JNI 头文件中声明的本地方法。
4. 编译本地代码：将本地代码编译成与目标平台兼容的动态链接库（例如，**`.dll`** 文件或 **`.so`** 文件）。
5. 在 Java 中调用本地方法：在 Java 代码中使用 **`native`** 关键字声明本地方法，并通过 JNI 接口调用本地方法。
通过 JNI，Java 程序可以获得更高的灵活性和性能，特别是在需要与底层系统交互或需要利用特定的本地功能时。然而，需要注意的是，过度使用 JNI 可能会增加代码的复杂性和可移植性，并且需要特别注意内存管理和安全性问题。
1. compareAndSwapInt(Object obj, long offset, int expect, int update)：
	- 功能：原子地将给定对象的偏移量处的整数值与期望值进行比较，如果相等，则更新为新值。
	- 参数：**`obj`**：目标对象、**`offset`**：偏移量，表示要修改的字段在对象中的偏移量、**`expect`**：期望值、**`update`**：更新值。
	- 返回值：如果更新成功，则返回 **`true`**，否则返回 **`false`**。
2. compareAndSwapLong(Object obj, long offset, long expect, long update)：
	- 功能：原子地将给定对象的偏移量处的长整型值与期望值进行比较，如果相等，则更新为新值。
	- 参数和返回值同上。
3. compareAndSwapObject(Object obj, long offset, Object expect, Object update)：
	- 功能：原子地将给定对象的偏移量处的对象引用与期望值进行比较，如果相等，则更新为新值。
	- 参数和返回值同上。
4. getInt(Object obj, long offset)：
- 功能：获取给定对象的偏移量处的整数值。
- 参数：**`obj`**：目标对象、**`offset`**：偏移量，表示要获取的字段在对象中的偏移量。
- 返回值：偏移量处的整数值。
5. putInt(Object obj, long offset, int value)：
	- 功能：设置给定对象的偏移量处的整数值为指定值。
	- 参数：**`obj`**：目标对象、**`offset`**：偏移量，表示要设置的字段在对象中的偏移量、**`value`**：要设置的整数值。
    
## 使用 Unsafe 类
Unsafe类是 Java 中的一个不稳定的、不建议直接使用的类，位于 sun.misc 包中。它提供了一些低级别的、直接操作内存和执行不安全的类型转换等操作的方法。这些操作通常是 Java 语言本身不允许的，因为它们可能会导致安全性问题和不可移植性。
	
所以一般是无法使用 Unsafe 类的，比如来看 Unsafe 类中提供的获取实例对象的方法
```java
	@CallerSensitive
	public static Unsafe getUnsafe() {
		Class<?> caller = Reflection.getCallerClass();
		if (!VM.isSystemDomainLoader(caller.getClassLoader()))
			throw new SecurityException("Unsafe");
		return theUnsafe;
	}
```

这个方法会去查看调用方方的类是否是由系统类加载器加载的，下面展示判断方法
```java
	/**
	 * Returns true if the given class loader is in the system domain
	 * in which all permissions are granted.
	 */
	public static boolean isSystemDomainLoader(ClassLoader loader) {
		return loader == null;
	}
```
当类加载器为 null，也就是使用 Bootstrap 类加载器加载，此时才可以调用类中的方法。
我们自己定义的类如果没有经过特殊处理，都是通过应用类加载器来加载的，此时自然无法通过这个方法来获取实例，此时可以通过 Java 中的后门，也就是反射来获取。
```java
public class TestCAS {
	public static void main(String[] args) throws NoSuchFieldException, NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {
		Class<String> stringClass = String.class;
		Field hash = stringClass.getDeclaredField("hash");
		hash.setAccessible(true);
		Class<Unsafe> unsafeClass = Unsafe.class;
		Constructor<?> declaredConstructor = unsafeClass.getDeclaredConstructor();
		declaredConstructor.setAccessible(true);
		Unsafe unsafe = (Unsafe)declaredConstructor.newInstance();
		Method objectFieldOffset = unsafeClass.getDeclaredMethod("objectFieldOffset", Field.class);
		objectFieldOffset.setAccessible(true);
		System.out.println(objectFieldOffset.invoke(unsafe, hash));
	}
}
```
上面代码中使用反射，成功获取了 String 类中的 hash 在内存中的偏移量。