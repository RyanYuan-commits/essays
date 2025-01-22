这里首先列出 ArrayList 扩容的几个特点，看完这些特点再去阅读体验会比较好
	1）ArrayList 中维护了一个 Object 类型的数组，elementData。
	2）当每次创建 ArrayList 对象的时候，如果使用的是无参构造器，则初始的 elementData 的容量为 0，第一次添加元素的时候则扩容 elementData 的容量为 10，如果需要再次扩容，则扩容为原来的 1.5 倍
	3）如果使用的是指定大小的构造器，则初始的 elementData 的容量就是指定的大小，如果需要扩容，也是直接扩容为 elementData 的 1.5 倍。
# 构造方法
```java
public ArrayList() {
	this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```
将 DEFAULTCAPACITY_EMPTY_ELEMENTDATA 赋值给了 elementData；
ArrayList 的内部实现就是基于这个 elementData 数组，它是 ArrayList 存放元素的位置，之后的拿取、扩容之类的操作本质上都是在操纵它。

```java
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
```
- DEFAULTCAPACITY_EMPTY_ELEMENTDATA 是一个共享的空数组实例，当使用 **默认无参构造方法** 时，elementData 会被赋值为 DEFAULTCAPACITY_EMPTY_ELEMENTDATA；
- 在使用有参构造方法的时候有参构造，如果将容量赋值为 0，elementData 会被赋值为 EMPTY_ELEMENTDATA，通过这种方式来区分是真的需要将容量设置为 0，还是说设置为默认值 10；
除了这个标志作用以外，它们的作用都是相同的，都是在内存中开辟了一个共享空间用来存放空数组，可以避免过早的为新创建的 ArrayList 实例分配内存，达到节省内存的作用。

```java
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+ initialCapacity);
        }
    }
    
    // 无参构造方法
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

```
如果指定的初始容量大于零，就直接构造一个容量为初始容量的 Object 数组，然后将其赋值给 elementData；
否则，当初始的容量等于 0 的时候，就指定其为 EMPTY_ELEMENTDATA，而不是上面的 DEFAULTCAPACITY_EMPTY_ELEMENTDATA；
如果是其他的数字，比如负数，就抛出一个异常，这也是一个常用的 Runtime Exception，IllegalArgumentException 标识方法入参异常。

```java
public ArrayList(Collection<? extends E> c) {
    Object[] a = c.toArray();
    if ((size = a.length) != 0) {
        if (c.getClass() == ArrayList.class) {
            elementData = a;
        } else {
            elementData = Arrays.copyOf(a, size, Object[].class);
        }
    } else {
        // replace with empty array.
        elementData = EMPTY_ELEMENTDATA;
    }
}
```
首先通过 Collction 接口中定义的 toArray() 方法，将集合类转为一个数组，方便后续的处理；
然后在 if 的条件中去指定 size（当前 ArrayList 实例中存放元素的个数）为数组的长度，然后判断这个长度是否为 0；
如果 c.getClass() 为 ArrayList 的话，就直接将数组 a 赋值给 elementData ，如果不是的话，需要再做一次转化，通过 Arrays.copyOf() 方法将数组复制为对象数组，这样做的原因有两个，
	一是为了保证类型安全问题；
	二是为了防止那个自定义的集合（为什么一定是自定义的？看看 Collection 接口的规定就知道了，遵守这个接口规范的一定不会将自己数组传入）头铁把自己内部的数组的引用传过来，导致操控的是他的数组。
# 底层扩容机制
```java
public boolean add(E e) {
	ensureCapacityInternal(size + 1);  // Increments modCount!!
	elementData[size++] = e;
	return true;
}
```
add 方法的代码其实比较简单，首先就是调用了 ensureCapacityInternal() 来确保存储空间（elementData）足够放下这个新的元素，然后再将元素放入其中：elementData\[size++] = e;

```java
private void ensureCapacityInternal(int minCapacity) {
	ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}
```
在这个方法中，首先是使用了 calculateCapacity(elementData, minCapacity) 方法去计算容量，我们首先考虑使用无参构造和初始容量为 0 的有参数构造，以及传入空集合的有参构造，但无论是哪种方法，最终形成的 ArrayList 实例的长度都是 0，所以此时的 minCapacity 就是 1。

```java
private static int calculateCapacity(Object[] elementData, int minCapacity) {
	if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
		return Math.max(DEFAULT_CAPACITY, minCapacity);
	}
	return minCapacity;
}
```
if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA)，判断这个 elementData 是否为 DEFAULTCAPACITY_EMPTY_ELEMENTDATA，也就是判断使用 无参构造方法 创建的实例。
当发现 ArrayList 实例是通过无参构造形成的，就会去取 minCapacity 和 DEFAULT_CAPACITY 中的最大值，而 DEFAULT_CAPACITY 它的值就是 10。

```java
private void ensureCapacityInternal(int minCapacity) {
	ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
  }

private void ensureExplicitCapacity(int minCapacity) {
	modCount++;
	// overflow-conscious code
	if (minCapacity - elementData.length > 0) grow(minCapacity);
}
```
首先是对这个 modCount 做自增操作，这个变量记录的是 ArrayList 集合扩容的次数。
然后去判断 minCapacity - elementData.length > 0 也就是最小需要的长度能否通过当前的 elementData 长度满足，如果不能就进入扩容方法 grow()，并且传入 minCapacity。

```java
private void grow(int minCapacity) {
		// overflow-conscious code
		int oldCapacity = elementData.length;
		int newCapacity = oldCapacity + (oldCapacity >> 1);
		if (newCapacity - minCapacity < 0)
			newCapacity = minCapacity;
		if (newCapacity - MAX_ARRAY_SIZE > 0)
			newCapacity = hugeCapacity(minCapacity);
		// minCapacity is usually close to size, so this is a win:
		elementData = Arrays.copyOf(elementData, newCapacity);
}
```
计算 newCapacity = oldCapacity + (oldCapacity >> 1); ， 得到 oldCapacity 原本长度的 1.5 倍，但是因为是最后将其转换为了 int，所以其扩容效果是小于等于 1.5 倍的；
然后将此时的 minCapacity 与 newCapacity 做比较，取最大值来作为下次扩容的大小；
最终就是调用 Arrays.copyOf(elementData, newCapacity); 将原本 elementData 中的内容移动到新拓展的长度为 newCapacity 的数组中，这就完成了一个完整的扩容；

对于无参构造和初始容量为 0 的有参数构造，以及传入空集合的有参构造，可以做如下的总结：
- 此时如果是无参构造，它带进来的 minCapacity 就是 10，最终其会被拓展为 10
- 如果是有参构造的话，带进来的 minCapacity 其实就是 1，且计算得 int newCapacity = oldCapacity + (oldCapacity >> 1); 结果是 0，那最终 elementData 会被拓展成 1。

所以说第一次拓展均拓展成 10 其实是不准确的；其他长度的拓展大家顺着流程推导一下就很容易得到了，画成图表就是这样的：

|minCapacity|newCapacity|最终值|
|---|---|---|
|1|0|1|
|2|1.5（1）|3|
|3|无需扩容|3|
|4|4.5（4）|4|
|5|6|6|
|6|无需扩容|6|
|7|9|9|