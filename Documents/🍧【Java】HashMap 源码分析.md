前导知识：[[🍜 红黑树]]
## 1 散列表与哈希算法
### 1.1 数组和链表
![[数组和链表.png|500]]
==数组==：数组是一种 **线性数据结构**，由一组连续的内存单元组成，每个元素都有固定的 **索引** 位置。
数组的优点是 **可以通过索引快速访问元素**，访问元素的时间复杂度为O(1)。
但是因为数组的长度是指定的，所以进行插入和扩容的操作非常麻烦，需要将原本数组中的内容转移到新数组中才能完成扩容的操作。

==链表==：链表也是一种线性数据结构，由一系列节点组成，每个节点包含一个数据元素和一个指向下一个节点的指针。
链表的优点是 **插入和删除操作** 可以在O(1)时间内完成，无需移动其他元素。
缺点是访问元素时需要 **遍历整个链表**，时间复杂度为O(n)，并且相比数组占用更多的内存空间。
### 1.2 散列表
![[散列表.png|500]]
与上面的两种数据结构不同的是，散列表是用来 **存储键值对** 的，在实现方式上，散列表像是数组 + 链表的一个结合，也就是基本的结构是一个数组，但数组中存储的元素是一个链表（本文仅讨论链地址法）。
提到数组，最显著的特点就是索引，散列表是通过 **哈希算法** 将键映射成一个长度固定的二进制数，然后通过 **路由算法** 将其映射到数组的索引中；
这样做了之后，如果我们在想要寻找这个被存储的元素，只需要通过相同的 key -> 哈希算法 -> 路由算法 的方式，就可以得到数组中的唯一一个索引。

通过这样的方式，散列表拥有了和链表一样快的插入速度，也有了可以媲美数组的查询速度，但是一切使用哈希算法的结构都不可避免的会遇到哈希冲突，即不同的 key 经过映射之后得到了相同的结果，需要解决冲突的方法，链地址法、开放散列法等；

在 HashMap 中使用的是优化之后的链地址法，即在数组的某个位置存储的是一个链表结构，如果遇到哈希冲突就将这些元素链接起来；与传统的链地址法不同的是，HashMap 的链地址法当某个位置链表过长的时候，会将这个链表转化为一个红黑树。
### 1.3 哈希算法
![[哈希算法.png|700]]
Hash 的中文释义为散列，一般音译为哈希。哈希算法的功能是将 **任意长度** 的输入，通过算法转变为固定长度的输出。
映射的规则称为 **哈希算法**，原始数据通过映射之后得到的二进制串就是 **哈希值**。

哈希算法有如下的几个特点：
- 无法通过哈希值反推出原始的数据，且数据一点微小的变化都会得到完全不同的哈希值，相同的数据会得到完全相同的哈希值，这两个特点使得哈希算法在安全方面有广泛的应用，比如 https 的数字证书的签名认证。
- 哈希算法的执行效率很高效，长文本也能很快的算出对应的哈希值。
- 由于是将任意长度是输出映射为固定长度的输出，将无限种数据映射为有范围的数据，必然会导致冲突，这也就是我们常说的 **哈希冲突**。如何处理哈希冲突是使用哈希函数的时候需要解决的问题。
## 2 HashMap 实现概览
### 2.1 数据结构
HashMap 是基于 **散列表** 实现的，其底层是通过数组+链表（Java8 引入了红黑树）来存储数据的。
HashMap 内部维护了一个数组，它是 HashMap 实例的一个成员变量，名为 table，这个数组的每个位置称为桶（Bucket）。 初始的数组长度如果不指定的话默认为 16，当插入元素过多的时候，HashMap 会使用一个更大的数组替换原来的 table，然后将原本 table 中的元素重新映射到这个新的数组。

Java 8 之后，HashMap 使用了数组 + 链表 + 红黑树的结构，以应对哈希冲突。每个桶可以存储一个链表或红黑树。
当哈希冲突发生时，新的键值对会被插入到对应位置的链表或红黑树中，具体来说，当某个链表的长度超过一定阈值（默认为8），且 table 的长度超过某个阈值（默认为 64）的时候，这个长链表会转换为红黑树，以提高查找、插入、删除操作的效率；红黑树是一种二叉查找树，它的查找时间复杂度为 O(log n) ，链表则是 O(n)。
### 2.2 路由算法
大致步骤是这样的：调用 key 的 hashCode 方法，key 对象映射为一个 32 位整数，为了尽可能减少哈希冲突，再对 hashCode 得到的哈希值进行一次扰动，作为最终的哈希值。
```java
// 扰动函数
static final int hash(Object key) {
	 int h;
	 return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// 路由计算
(table.length - 1) & node.hash
```
### 2.3 Node 节点
Node 将 value 做一层包装，存储一些其他重要信息：
```java
/**
 * 基本的哈希节点，适用于大部分的实例，HashMap 的红黑树节点 TreeNode、
 * LinkedHashMap 中的节点 Entry 都是基于这个 Node 实现的
 */
static class Node<K,V> implements Map.Entry<K,V> {
	// key 的 hash
	final int hash;
	// key 本身
	final K key;
	// value 本身
	V value;
	// 下一个节点的指针（链表）
	Node<K,V> next;

	Node(int hash, K key, V value, Node<K,V> next) {
		this.hash = hash;
		this.key = key;
		this.value = value;
		this.next = next;
	}
}
```

当 Bucket 中存储的是链表的时候，使用的节点类是 Node，而当存储的是红黑树的时候，使用 TreeNode 作为构成元素：
```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {  
    TreeNode<K,V> parent;
    TreeNode<K,V> left;  
    TreeNode<K,V> right;  
    TreeNode<K,V> prev; 
    boolean red;
    
    // ......
}
```
TreeNode 也是 HashMap 的静态内部类，它继承自 LinkedHashMap.Entry，TreeNode 除了作为红黑树的节点以外，还作为一个双向的链表节点；
TreeNode 在 Node 的基础上拓展了 parent、left、right、red 这些红黑树相关的属性，以及 prev 这个指向前驱节点，用于维护双向链表的属性。
## 3 源码阅读
### 3.1 关键常量解读
在正式阅读关键源码之前，我们先从了解 HashMap 中的关键常量开始，这些关键常量和 HashMap 的基本结构，以及扩容、树化等特殊机制息息相关，掌握它们对于理解源码很有帮助。

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
```
- 如果在实例化 HashMap 的时候没有指定初始容量，其默认值为 16。

```java
static final int MAXIMUM_CAPACITY = 1 << 30;
```
- 表示 table 的最大容量，限定了一个扩容的上限，在后续扩容的时候，如果发现要扩容到比上限还要大，就将容量先扩容到上限，如果在容量已经达到上限后再次执行扩容方法的话，则不会继续扩容。

```java
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```
- HashMap 默认的负载因子的值，负载因子（Load Factor）是指哈希表中已存储元素数量与哈希表容量的比例。
- 在哈希表中，负载因子用于衡量哈希表的填充程度，即已存储元素占哈希表容量的比例，当已存储元素占比超过某个阈值的时候，哈希冲突发生概率会大大提升，此时就需要进行扩容操作，以维持一个比较健康的状态。
- HashMap 插入元素后会判断 table 中的元素个数是否大于 `capacity * loadFactor`，如果是，就会触发扩容机制。常用的负载因子值为 0.75，这是一个经验值，可以在平衡内存占用和性能之间做出权衡，非必要不需要自己额外指定。

```java
static final int TREEIFY_THRESHOLD = 8;

static final int UNTREEIFY_THRESHOLD = 6;

static final int MIN_TREEIFY_CAPACITY = 64;
```
- 上面的三个常量都和 HashMap 的树化机制有关，`TREEIFY_THRESHOLD` 和 `MIN_TREEIFY_CAPACITY` 决定了链表树化的时间，具体来说，当一个链表的长度达到 8 且此时 table 数组的长度达到 64，这个链表会转化为红黑树结构；
- 之所以也要对 table 的长度做限定，是因为当 table 长度过小的时候，发生哈希冲突的概率很大，链表长度会很容易达到阈值，这时候执行树化操作性价比较低，频繁树化操作会影响性能。
- `UNTREEIFY_THRESHOLD` 是树降级为链表的阈值，当树中的元素降低到 6 个后，将树降级为链表。
### 3.2 关键属性解读
```java
transient Node<K,V>[] table;
```
HashMap 底层中实际存储数据的容器，是一个 Node 数组；table 被 transient 关键字修饰，在 HashMap 实例对象序列化的时候 table 不会参与序列化，因为 table 除了存储 k-v 之外，还维护了链表、红黑树的结构，这些结构性的内容传输过去并没有什么作用；
HashMap 提供了 writeObject 和 readObject 方法来进行序列化和反序列化，传输过程中只需要传输键值对，而不是整个 table。

```java
transient int size;
```
当前 Map 中存储的 K-V 键值对的数量。

```java
int threshold;
```
扩容阈值，具体值为 capacity * loadFactor，当 Map 中的元素个数达到 threshold 就会触发扩容。

```java
final float loadFactor;
```
负载因子的值，可以通过 HashMap(int initialCapacity, float loadFactor) 构造方法指定。

### 3.3 构造方法
#### 3.3.1 指定容量和负载因子
```java
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
```
- 当发现初始容量大于容量上限时，会先将 initialCapacity 设为 MAXIMUM_CAPACITY。
- 调用 `tableSizeFor` 方法用于计算 table 最终容量，因为 HashMap 的数组容量一定是 2 的幂，然后将结果赋值给 `this.threshold`，当创建 table 数组的时候，会将 threshold 作为容量。
#### 3.3.2 指定初始容量
```java
public HashMap(int initialCapacity) {
	this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
```
指定初始容量的构造方法，直接调用了前面的 HashMap(int initialCapacity, float loadFactor) 方法。 
#### 3.3.3 空参构造
```java
public HashMap() {
	this.loadFactor = DEFAULT_LOAD_FACTOR;
}
```
- 为负载因子赋默认值 `DEFAULT_LOAD_FACTOR`
- 方法中没有指定 threshold 值，HashMap 将 threshold 为零作为用户未指定初始容量的标识，在创建 table 的时候，如果发现 threshold 为 0，就会将 table 的容量设置为默认值 16。
#### 3.3.4 基于其他 Map 构造
```java
public HashMap(Map<? extends K, ? extends V> m) {
	this.loadFactor = DEFAULT_LOAD_FACTOR;
	putMapEntries(m, false);
}
```
调用了 putMapEntries 方法，将传入 Map 中的数据插入到本 HashMap 中：
```java
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {  
    int s = m.size();  
    if (s > 0) {  
        if (table == null) { // pre-size  
            float ft = ((float)s / loadFactor) + 1.0F;  
            int t = ((ft < (float)MAXIMUM_CAPACITY) ? (int)ft : MAXIMUM_CAPACITY);  
            if (t > threshold)  threshold = tableSizeFor(t);  
        }  
        else if (s > threshold) resize();  
        
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {  
            K key = e.getKey();  
            V value = e.getValue();  
            putVal(hash(key), key, value, false, evict);  
        }  
    }  
}
```
- 计算一个合理的容量，将其保存到 threshold 中；
- 而当 table 不为 null 的时候，如果发现需要插入的元素个数已经达到的自己的扩容阈值，预先进行一次扩容；
- 遍历传入的 Map，调用 putVal 方法将 K-V 键值对一个一个插入。

---

通过上面的四个构造方法，我们可以得出这么两个结论：
- HashMap 为了优化内存，将 table 的初始化时机设计到首次插入元素后；
- 使用 HashMap#tableSizeFor 方法保证了 table 的容量一定是 2 的幂次。

### 3.4 putVal 方法
当我们想要向 HashMap 中插入元素的，会调用 put 方法，这个方法底层调用的是 putVal 方法：
```java
 public V put(K key, V value) {
	 return putVal(hash(key), key, value, false, true);
}

static final int hash(Object key) {
	 int h;
	 return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
在调用 putVal 之前，会调用 hash 方法计算 key 的哈希值：
- 如果传入的 key 是 null，那就直接映射为 0，从这里可以看出 ==HashMap 是可以使用 null 作为 key 的==；
- 如果 key 不为 null，执行 hashCode 方法获得哈希值，然后将这个值与其右移 16 位的值做按位异或运算。

>[!question] 为什么不直接采用原始的哈希值呢？
> HashMap 的路由算法是这样的：`(table.length - 1) & node.hash；`，数组的长度越小，key 起作用的哈希值的位数就越少，不同 key 之间的区分度就比较低，也就容易出现哈希冲突；
> 为了让不同元素哈希值的分布尽量分散，可以让高位的值也参与进来，这种操作很形象的被称为 “扰动”，具体的实现方式被称为 ”扰动函数“。

```java
 final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
	 // 底层 table
	 Node<K,V>[] tab; 
	 // i 位置此时的元素，null 表示没有元素
	 Node<K,V> p;
	 // i: node 节点插入的位置
	 // n: table 数组的长度
	 int n, i;
	 if ((tab = table) == null || (n = tab.length) == 0) n = (tab = resize()).length;
	 
	 if ((p = tab[i = (n - 1) & hash]) == null) tab[i] = newNode(hash, key, value, null);
	 else { 
		// 插入位置有元素
	 }
	 // 被修改的次数
	 ++modCount;
	 // size 达到扩容阈值，扩容
	 if (++size > threshold) resize();
	 afterNodeInsertion(evict);
	 return null;
 }
```
- 先判断 table 数组是否被创建（`table == null || tab.length == 0` ），如果还没有，执行 resize 方法来创建 table，==HashMap 将 table 的创建延后到第一次插入元素时==。
- 然后判断插入位置是否有元素（`tab[(n - 1) & hash] == null`），若为 null 则直接将新节点插入。
- 当插入位置有元素的时候处理方式是这样的：
```java
else {
	// 如果插入的 key 存在于 table 中，e 表示该 Node，否则，e 为 null
	Node<K,V> e;
	// 暂存插入位置元素的 key 值
	K k;
	// 插入位置的 Bucket 中首个元素的 key 和 Node 的 key 相同
	if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k)))) e = p;
	
	// 插入位置的 Bucket 中存储的是红黑树
	else if (p instanceof TreeNode) e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
	
	// 插入位置的 Bucket 中存储的是链表
	else { 
		for (int binCount = 0; ; ++binCount) {
			if ((e = p.next) == null) { // 遍历过程中没有找到与插入节点 key 相同的情况
				p.next = newNode(hash, key, value, null);
				if (binCount >= TREEIFY_THRESHOLD - 1) treeifyBin(tab, hash);
				break;
			}
			 // 遍历节点的时候找到了与插入节点 key 相同的节点
			if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))) break;
			p = e;
		}
	}
	// 针对替换操作的逻辑
	if (e != null) { 
		V oldValue = e.value;
		if (!onlyIfAbsent || oldValue == null)
			e.value = value;
		afterNodeAccess(e);
		return oldValue;
	}
}
```
- 遍历 Bucket 中存储的元素，如果 key 已存在，使用变量 e 来暂存这个 Node 的引用，后续执行替换操作；
- 如果此时 Bucket 中存储的是一个红黑树，调用 putTreeVal 方法来处理。
- 如果 key 是第一次出现，也就是在扫描 Bucket 的时候发现 `p.next == null`，直接将新的 Node 置于链表结尾。

当插入 key 之后，会检查是否需要执行树化：
```java
final void treeifyBin(Node<K,V>[] tab, int hash) {  
    int n, index; Node<K,V> e;  
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)  
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {  
		// 树化
    }  
}
```
### 3.5 resize 方法
resize 可以在 table 未创建的时候被调用，用来创建 table，也可以对已创建的 table 进行扩容，方法总体可以分为两部分，第一部分是确定新的容量和新的扩容阈值，第二部分是执行实际的扩容操作：
#### 3.5.1 确定新的容量和扩容阈值
```java
final Node<K,V>[] resize() {
	// 第一部分：确定 newCap 和 newThr 的值
	Node<K,V>[] oldTab = table;
	int oldCap = (oldTab == null) ? 0 : oldTab.length;
	int oldThr = threshold;
	int newCap, newThr = 0;
	
	// 如果 table 已经被创建
	if (oldCap > 0) {
		if (oldCap >= MAXIMUM_CAPACITY) {
			threshold = Integer.MAX_VALUE;
			return oldTab;
		}
		// 将容量扩容为原来的二倍
		else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY) newThr = oldThr << 1;
	}
	// 初始容量存储在 threshold 中
	else if (oldThr > 0) newCap = oldThr;
	// threshold 标识着使用默认的容量
	else {               
		newCap = DEFAULT_INITIAL_CAPACITY;
		newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
	}
	if (newThr == 0) {
		float ft = (float) newCap * loadFactor;
		newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ? (int)ft : Integer.MAX_VALUE);
	}
	threshold = newThr;
	
	// ......
}
```
标准情况下，执行流程是这样的：
- 如果 table 已经被创建（`oldCap > 0`），此时执行的是扩容：
	- 判断容量是否超过扩容上限 `MAXIMUM_CAPACITY`，如果是就直接返回；
	- 计算 `oldCap << 1` 的值，将其赋值给 newCap，同时将 oldThr 左移一位，赋值给 newThr。
- 当 table 的未被创建时：
	- 如果 oldThr 不为 0，表明用户指定了初始容量，将其赋值给 newCap，否则，使用默认容量
	- 使用默认的负载因子计算出扩容阈值，赋值给 newThr。
#### 3.5.2 创建新 table 并迁移
```java
// 第二部分：将原数组中内容复制到拓展后的新数组中
@SuppressWarnings({"rawtypes","unchecked"})
Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
table = newTab;
if (oldTab != null) {
	for (int j = 0; j < oldCap; ++j) { // 遍历原本的数组
		Node<K,V> e;
		if ((e = oldTab[j]) != null) {
			oldTab[j] = null;
			// 仅有一个节点的情况
			if (e.next == null) newTab[e.hash & (newCap - 1)] = e;
			 // 是树节点的情况
			else if (e instanceof TreeNode)((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
			else { 
				// 对链表迁移的优化
			}
		}
	}
}
return newTab;
```
- 首先根据上面计算出来的新的容量，创建新的 table 数组；
- 然后如果老 table 不为 null，就需要执行复制转移的操作，使用一个 for 循环来遍历老 table 的所有位置并执行迁移；
#### 3.5.3 对链表迁移的优化
对于链表节点，HashMap 在迁移的时候进行了优化：
```java
Node<K,V> loHead = null, loTail = null;
Node<K,V> hiHead = null, hiTail = null;
Node<K,V> next;
do {
	next = e.next;
	if ((e.hash & oldCap) == 0) {
		if (loTail == null) loHead = e;
		else loTail.next = e;
		loTail = e;
	}
	else {
		if (hiTail == null) hiHead = e;
		else hiTail.next = e;
		hiTail = e;
	}
} while ((e = next) != null);
if (loTail != null) {
	loTail.next = null;
	newTab[j] = loHead;
}
if (hiTail != null) {
	hiTail.next = null;
	newTab[j + oldCap] = hiHead;
}
```
这些节点处于相同的 Bucket，也就表明它们 `node.hash & oldTab.length - 1`  的结果相同；

这些节点在新的链表中的索引，也就是 `hash & oldTab.length*2 - 1` 的值是有迹可循的：
![[HashMap 扩容优化.png|500]]
- 以 16 拓展到 32 为例，刨去相同的部分，哈希的前缀要么是 1 要么是 0
- 如果前缀为 1，与 32 - 1 做按位与之后，得到的结果就是原本的位置加上一个 16，也就是 oldTab.length
- 而对于前缀为 0 的元素，位置则不会改变；
将位于原本位置 + oldTab.length 的元素称为高位元素，将仍处于原本位置的命名为低位元素；由此引出了下面的四种变量：
```java
Node<K,V> loHead = null, loTail = null; // 需要放置在新 table 低位节点的头部和尾部
Node<K,V> hiHead = null, hiTail = null; // 需要放置在新 table 高位的节点的头部和尾部
```
在迁移的时候，构造出低位和高位两个链表，再存入新 table 的 Bucket 中，大大优化了迁移的效率。
## 4 HashMap 的线程安全问题
### 4.1 多线程 put 导致的元素丢失问题
如果多个线程同时向 HashMap 中添加元素，会导致元素丢失的问题，我们根据源码来分析一下：
```java
else { 
	for (int binCount = 0; ; ++binCount) {
		if ((e = p.next) == null) { // 遍历过程中没有找到与插入节点 key 相同的情况
			p.next = newNode(hash, key, value, null);
			if (binCount >= TREEIFY_THRESHOLD - 1) treeifyBin(tab, hash);
			break;
		}
		 // 遍历节点的时候找到了与插入节点 key 相同的节点
		if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))) break;
		p = e;
	}
}
if (e != null) { // 针对替换操作的逻辑
	V oldValue = e.value;
	if (!onlyIfAbsent || oldValue == null)
		e.value = value;
	afterNodeAccess(e);
	return oldValue;
}
```
上面截取的是 HashMap 插入的时候如果插入的位置有元素的情况；
我们假设此时这个位置有一个元素 k1：
![[线程 put 导致的元素丢失问题1.png|100]]
此时一个线程插入 k2，另一个线程插入 k3；假设两个线程都判断 if ((e = p.next) == null) 为 true，进入到这个代码段：
```java
p.next = newNode(hash, key, value, null);
if (binCount >= TREEIFY_THRESHOLD - 1) treeifyBin(tab, hash);
break;
```
此时线程 1 插入了 k2，结构如下图所示
![[多线程 put 导致的元素丢失问题 2.png|300]]
但是线程 2 仍然会执行 `p.next = newNode(hash, key, value, null);`，此时 k2 就丢失了，这就是多线程 put 导致的元素丢失问题。
### 4.2 put 和 get 并发的时候可能导致 get 为 null
线程 1 执行 put 时，因为元素个数超出 threshold 而导致扩容，线程 2 此时执行 get，有可能导致这个问题。
```java
Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
table = newTab;
```
这是因为在 resize 扩容方法中，是先将 table 设置为 newTab，然后再复制的；
此时如果有另一个线程调用 get 方法，而新的数组没有被填充完，那此时这个线程得到的就是值。
### 4.3 JDK7 中并发 resize 会造成循环链表
JDK 7 对 resize 方法在并发条件下，可能会产生循环链表，这个问题在 JDK 8 已经得到了解决，JDK7 中的 resize 方法是这样的：
```java
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }

    Entry[] newTable = new Entry[newCapacity];
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
    table = newTable;
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}

// 关键在于这个 transfer 方法，这个方法的作用是将旧 hash 表中的元素 rehash 到新的hash表中
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            // 用元素的 hash 值计算出这个元素在新hash表中的位置
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[I];
            newTable[i] = e;
            e = next;
        }
    }
}
```
JDK 7 的扩容，采用的是头插法，也就是将新加入的数据，作为 Bucket 中存储的第一个数据，下图中展示的是一个扩容操作，插入的顺序是 A、B、C，但扩容之后的结果顺序是 C、B、A
![[头插法.png|700]]
其具体逻辑表现在这段代码上，e 是要插入新 table 的节点：
```java
e.next = newTable[I];
newTable[i] = e;
```
- 假设此时有一个线程 1，它的局部变量表中的 e 存储的是 A 节点，next 存储的是 B 节点，然后其被调度下线；
- 此时线程 2，也在执行 resize 方法，它将 A、B、C 插入到新的链表中；
- 此时线程 1 被调度上线，它仍认为此时还未进行转移，依然执行，e.next = newTable\[I]; 此时就将 A 指向了 C;
- 然后移动指针，此时 e 为 B，next 为 null，再次执行 `e.next = newTable[I]`，这时候又将 B 指向了 C，循环结束：
![[JDK7 中并发 put 会造成循环链表.png|500]]
这时候，一个循环链表就产生了，产生这个问题的根本原因就是 JDK 7 多个线程在插入过程中的并发操控 newTable，因为其在新的数组中的顺序和旧的数组中是不同的，会导致指针的回连。

在 JDK 8 的更新中，HashMap 选择在每个线程内部构建新的两个链表，插入的时候将这些构建好的链表整体插入，各个线程之间构造的链表互不影响，且均为正确的，所以不管最终数组存储的是哪个线程构建的链表，结果都是正确的。
```java
do {
	next = e.next;
	if ((e.hash & oldCap) == 0) {
		if (loTail == null) loHead = e;
		else loTail.next = e;
		loTail = e;
	}
	else {
		if (hiTail == null) hiHead = e;
		else hiTail.next = e;
		hiTail = e;
	}
} while ((e = next) != null);
```
---

JDK 7 中，由于所有线程在扩容的过程中，均是不断操纵和更新 newTable，导致了成环的问题；
JDK 8 之后，是先将链表构造好，再插入，此时并发的插入并不会造成最终结果的错误。




