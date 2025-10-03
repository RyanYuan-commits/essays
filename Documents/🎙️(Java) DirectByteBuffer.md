---
type: Java
sub-type: 并发编程
finished: "false"
---
`DirectByteBuffer` 是 Java 用于实现**堆外内存**的一个重要的类, 通常用于在通信过程中来做缓冲池, 在 Netty, MINA 等 NIO 框架中使用更广泛;

## 1 Unsafe 管理堆外内存

其中对于堆外内存的创建, 使用, 销毁等逻辑均使用 [[🍦(Java) Unsafe|Unsafe]] 类提供的堆外内存 API 来实现.

```java
DirectByteBuffer(int cap) {
    super(-1, 0, cap, cap, null);  
    boolean pa = VM.isDirectMemoryPageAligned();  
    int ps = Bits.pageSize();  
    long size = Math.max(1L, (long)cap + (pa ? ps : 0));  
    Bits.reserveMemory(size, cap);  
  
    long base = 0;  
    try {  
	    // 1 分配内存并返回基地址
        base = UNSAFE.allocateMemory(size);  
    } catch (OutOfMemoryError x) {  
        Bits.unreserveMemory(size, cap);  
        throw x;  
    }  
    // 2 内存初始化
    UNSAFE.setMemory(base, size, (byte) 0);  
    if (pa && (base % ps != 0)) {  
        address = base + ps - (base & (ps - 1));  
    } else {  
        address = base;  
    }  
    try {
	    // 3 跟踪 DirectByteBuffer 对象的垃圾回收来实现堆外内存的释放  
        cleaner = Cleaner.create(this, new Deallocator(base, size, cap));  
    } catch (Throwable t) {  
        UNSAFE.freeMemory(base);  
        Bits.unreserveMemory(size, cap);
        throw t;
    }
    att = null;
}
```

## 2 Cleaner 实现内存释放

`Cleaner` 继承自虚引用 [[💟(Java) 引用类型#4 虚引用 PhantomReference|PhantomReference]], 当某个被 `Cleaner` 引用的对象将被回收时, JVM 会将此对象的应用放到 `pending` 链表中, 等待 `Reference-Handler` 进行相关处理;

`Reference-Hander` 是拥有最高优先级的守护线程, 会循环不断的处理 `pending` 链表中的对象引用, 执行 `Cleaner` 的清理方法.

```java
// java.nio.DirectByteBuffer.Deallocator#run: 释放内存
public void run() {  
    if (address == 0) {  
        // Paranoia  
        return;  
    }  
    UNSAFE.freeMemory(address);  
    address = 0;  
    Bits.unreserveMemory(size, capacity);  
}

// java.lang.ref.Reference#processPendingReferences: ReferenceHandler 执行的代码
private static void processPendingReferences() {
	// ......
	
	while (pendingList != null) {
		Reference<?> ref = pendingList;
		pendingList = ref.discovered;
		ref.discovered = null;

		if (ref instanceof Cleaner) {
			// 运行 Cleaner 的 clean 方法, 最终调用到上面的方法
			((Cleaner)ref).clean();
			synchronized (processPendingLock) {
				processPendingLock.notifyAll();
			}
		} else {
			ref.enqueueFromPending();
		}
	}
	
	// ......
}
```