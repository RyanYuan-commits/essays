`ThreadLocal` ，从字面含义上来看就是 "线程本地"，这很形象的说明了它的用处；
`ThreadLocal` 是 Java 提供的一种线程本地存储机制，可以利用该就只将数据还存在某个线程的内部，该线程可以在任意的时刻、任意的方法中去获取缓存的数据。
### 1 总体介绍
`ThreadLocal` 共有两种类型， `ThreadLocal` 本身和他的子类 `InheritableThreadLocal`。
```java
ThreadLocal<String> threadLocalString = new ThreadLocal<>();
ThreadLocal<Integer> threadLocalInteger = new ThreadLocal<>();
```
`ThreadLocal` 可以用于在每个线程中存储不同类型的数据，并且每个线程访问该`ThreadLocal` 对象时，都可以获取到其线程私有的变量副本。

`InheritableThreadLocal` 类型的 `ThreadLocal`： `InheritableThreadLocal` 是 `ThreadLocal` 的一个子类；它允许子线程访问父线程设置的本地变量，具体原理是：当子线程创建时，它会继承父线程中 `InheritableThreadLocal` 的值。例如：
```java
InheritableThreadLocal<String> inheritableThreadLocal = new InheritableThreadLocal<>();
```
### 2 ThreadLocal
#### 2.1 基本介绍
`ThreadLocal` 底层是通过 `ThreadLocalMap` 实现的， `ThreadLocalMap` 是 `Thread` 类的属性：
```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```
`ThreadLocalMap` 是 `ThreadLocal` 的静态内部类，存储的 KEY 为 `ThreadLocal` 对象，`VALUE` 就是实际的数据。
#### 2.2 ThreadLocal 的 set 方法
```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value);
    }
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```
- 首先，获取到当前执行该方法的线程，通过调用 `getMap(Thread t)` 方法来获取线程的 `threadLocals` 属性。
- 线程对象的 `threadLocals` 属性初始为 `null`，它正式的创建时机就是第一次调用 `ThreadLocal` 的 `set()` 方法的时候，所以可以看到上面还有一个 `createMap` 方法;
- 这里要特别关注键值对的内容：键是 `this` 也就是 `ThreadLocal` 对象本身，而值就是上面传入的 `value`，从这里可以看出，`ThreadLocal` 是作为键值对的键存在的，真实的数据其实就是存储在线程对象 `Thread` 中的。
#### 2.3 ThreadLocalMap
>既然谈到了 `ThreadLocalMap` 类，这里来详细的看看它是如何实现键值对的对应关系的；
>以及在 Entry 如何通过虚引用（WeakReference）来解决内存泄露的问题的？
##### Entry 对象
`ThreadLocalMap` 是 `ThreadLocal` 的一个静态内部类，数据被封装成一个 `Entry` 类，然后存储在 `ThreadLocalMap` 的 `table` 属性中：
```java
/**
 * The table, resized as necessary.
 * table.length MUST always be a power of two.
 */
private Entry[] table;
```

`Entry` 是 `ThreadLocalMap` 的一个内部类：
```java
static class Entry extends WeakReference<ThreadLocal<?>> {
	/** The value associated with this ThreadLocal. */
	Object value;

	Entry(ThreadLocal<?> k, Object v) {
		super(k);
		value = v;
	}
}
```

我们将它和 `HashMap` 中对数据的封装类 `Node` 做一下比较：
```java
static class Node<K,V> implements Map.Entry<K,V> {  
    final int hash;  
    final K key;  
    V value;  
    Node<K,V> next;
    // 其他方法......
}
```
与 `Node` 不同的是，`Entry` 并没有直接保存对 KEY 的引用，而是通过继承 `WeakReference` 实现了对 KEY 的虚引用。
而当对象 **仅** 被虚引用所指向的时候，JVM 进行 GC 垃圾回收的时候，**会直接将其回收**，也就是当 `ThreadLocal` 对象仅仅被 `ThreadLocalMap` 中的 `Entry` 引用的时候，`ThreadLocal` 对象仍然会被回收。
但是 `Thread` 对象中的 `Node` 是不会随着 `ThreadLocal` 的回收而自动回收的，因为 `table` 数组对 `Entry` 对象的引用属于强引用，这就引发了内存泄露的问题，也就是使用的内存并没有被及时的回收。
##### 清除无效 Entry 对象
如果这样看的话，理论上只要线程对象不被销毁，那 `Entry` 对象就永远不会被回收，这设计显然是不太合理的，所以 `ThreadLocalMap` 中还存在这样一些自动回收方法，其中比较常用的是 `cleanSomeSlots`：
```java
private boolean cleanSomeSlots(int i, int n) {  
    boolean removed = false;  
    Entry[] tab = table;  
    int len = tab.length;  
    do {  
        i = nextIndex(i, len);  
        Entry e = tab[i];  
        // 检测 Entry 是否引用了 null
        if (e != null && e.refersTo(null)) {  
            n = len;  
            removed = true;  
            i = expungeStaleEntry(i);  
        }  
    } while ( (n >>>= 1) != 0);  // 对数级别的时间复杂度
    return removed;  
}
```
以大约 `O(logn)` 的时间复杂度去检测数组中元素是否存在空引用（也就是检测 `ThreadLocal` 是否被回收），如果出现了空引用，就调用 `expungeStaleEntry(i)` 方法；
这个方法的效果是将索引位置的条目设置为 null，并减少哈希表的大小；并且，从索引位置开始，遍历到下一个 null 条目之前的所有条目，清除其他失效条目，并 **重新计算** 有效条目的哈希值，将其放置在正确的位置。
上面提到，重新计算的原因是，`ThreadLocalMap` 解决哈希冲突的方法是链地址法，如果后面的元素很可能原本的索引位置是前面删除的元素，这样可以再每次删除元素的时候规整数组。
这个方法的执行时机有这么两个：
- `set(ThreadLocal<?> key, Object value)`：调用 set 添加 K-V 键值对的时候
- `replaceStaleEntry(ThreadLocal<?> key, Object value,  int staleSlot)`：当插入元素的时候，如果发现该位置 `Entry` 对象的引用为 null，表示这个 `Entry` 已经失效，此时会调用这个方法来进行替换操作，此时需要再检测一下是否有其他 `Entry` 过期。
但这些自动回收方式都是相对被动的，最根本的方式还是要在 `ThreadLocal` 销毁前调用它的 `remove()` 方法：
```java
public void remove() {  
    ThreadLocalMap m = getMap(Thread.currentThread());  
    if (m != null) {
        m.remove(this);  
    }
}
```
##### 哈希函数与线性探测法
`ThreadLocalMap` 计算索引的方式为：
```java
int i = key.threadLocalHashCode & (len-1);
```
直接将 `ThreadLocal` 对象的 `HashCode` 与 数组的长度减一 做一个按位与的操作，将其映射到数组中的一个位置。
在 `ThreadLocalMap` 中，采用了**线性探测的方式**来解决哈希冲突，当出现了哈西冲突的时候，具体体现在 `nextIndex` 方法中：
```java
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}
```
在原位置的基础上不断加一来寻找下一个存放位置，当越界的时候就再从 0 开始寻找。
##### getEntry 方法
```java
private Entry getEntry(ThreadLocal<?> key) {  
    int i = key.threadLocalHashCode & (table.length - 1);  
    Entry e = table[i];  
    if (e != null && e.get() == key)  
        return e;  
    else  
        return getEntryAfterMiss(key, i, e);  
}
```
上面的是 `ThreadLocalMap` 的 `get` 方法，其通过哈希函数获取到坐标后，从 `table` 中获取并返回 `Entry`。
因为 `ThreadLocalMap` 采用的是链地址法，只探测一个位置没有数据显然是不够的，`getEntryAfterMiss` 方法会一直检索到下一个空位置，如果还没有找到对应的元素才会返回 `null`。
##### set 方法
```java
private void set(ThreadLocal<?> key, Object value) {
	Entry[] tab = table;
	int len = tab.length;
	int i = key.threadLocalHashCode & (len-1);

	for (Entry e = tab[i];
		 e != null;
		 e = tab[i = nextIndex(i, len)]) {
		ThreadLocal<?> k = e.get();

		if (k == key) {
			e.value = value;
			return;
		}

		if (k == null) {
			replaceStaleEntry(key, value, i);
			return;
		}
	}

	tab[i] = new Entry(key, value);
	int sz = ++size;
	if (!cleanSomeSlots(i, sz) && sz >= threshold)
		rehash();
}
```
不断通过 `nextIndex` 方法进行地址探测，找到空位置或相同的 KEY 执行替换或插入操作。
#### 2.4 ThreadLocal 的 get 方法
get 方法是以当前的 `ThreadLocal` 对象作为 KEY，从 Thread 对象的 `ThreadLocalMap` 中去取得对应的 VALUE 值：
```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```
通过 `getMap()` 方法获取线程对象的 `ThreadLocalMap` ，然后通过 `getEntry()` 方法从 map 中获取对应的数据，也就是通过 `ThreadLocal` 的哈希值，去映射到`ThreadLocalMap` 的 Entry 数组的一个下标，来将对应的对象取出。
### 3 InheritableThreadLocal
继承关系：
![[InheritableThreadLocal.png|500]]
`InheritableThreadLocal` 继承了父类 `ThreadLocal` 并且重写了它的 `getMap()` 和 `createMap()` 等方法
```java
    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
```
#### 3.1 set 方法
```java
public void set(T value) {
	Thread t = Thread.currentThread();
	ThreadLocalMap map = getMap(t);
	if (map != null) {
		map.set(this, value);
	} else {
		createMap(t, value);
	}
}
```
此时通过 `getMap()` 获取到的不再是该线程的 `threadLocals` 而是 `inheritableThreadLocals`。
#### 3.2 创建子线程
创建子线程时，父线程如果发现自己的 `InheritableThreadLocal` 不为空，会将里面的内容拷贝到字线程的同属性中。
```java
private void init(ThreadGroup g, Runnable target, String name,
				  long stackSize, AccessControlContext acc,
				  boolean inheritThreadLocals) {
	// ......
	Thread parent = currentThread();
	if (inheritThreadLocals && parent.inheritableThreadLocals != null)
		this.inheritableThreadLocals =
			ThreadLocal.createInhritedMap(parent.inheritableThreadLocals);
			
		  // 。。。。。。
}
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
	return new ThreadLocalMap(parentMap);
}
```
线程初始化的时候，父线程如果发现自己的 `inheritableThreadLocals` 存在值，就会调用 `ThreadLocal` 中的 `createInhritedMap` 方法；

这个方法是静态内部类 `ThreadLocalMap` 的 `private` 构造方法，它负责将参数中的 `ThreadLocalMap` 复制一份到当前创建的对象之中：
```java
private ThreadLocalMap(ThreadLocalMap parentMap) {
	Entry[] parentTable = parentMap.table;
	int len = parentTable.length;
	setThreshold(len);
	table = new Entry[len];

	for (int j = 0; j < len; j++) {
		Entry e = parentTable[j];
		if (e != null) {
			@SuppressWarnings("unchecked")
			ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
			if (key != null) {
				Object value = key.childValue(e.value);
				// 是复制一份，而不是直接存储
				Entry c = new Entry(key, value);
				int h = key.threadLocalHashCode & (len - 1);
				while (table[h] != null)
					h = nextIndex(h, len);
				table[h] = c;
				size++;
			}
		}
	}
}
// 控制子线程的继承内容  
protected T childValue(T parentValue) {
	return parentValue;
}
```
复制的过程是深拷贝(`Entry c = new Entry(key, value);`)，这样，子线程就继承了父亲线程的的 `InheritableThreadLocal` 中存储的内容。