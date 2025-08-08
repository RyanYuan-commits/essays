==ArrayList 扩容的特点==
- ArrayList 中实际负责存储的结构是一个名为 elementData 的 Object 数组。
- 当每次创建 ArrayList 对象的时候，如果使用的是无参构造器，初始的容量为 10，后续扩容会变为原来的 1.5 倍。
- 如果使用的是指定大小的构造器，则初始的 elementData 的容量就是指定的大小，如果需要扩容，也是扩容为原来的的 1.5 倍。
### 1 构造方法
#### 1.1 无参构造
```java
public ArrayList() {
	this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```
将 elementData 赋值为 DEFAULTCAPACITY_EMPTY_ELEMENTDATA，这是个空数组。
除了这个常量以外，ArrayList 还有一个名为 EMPTY_ELEMENTDATA 的常量，它也是一个空数组，当用户指定初始容量为 0 的时候，elementData 会被赋值为 EMPTY_ELEMENTDATA，起到一个标识作用。

#### 1.2 指定初始容量构造
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
- 如果指定的初始容量大于零，就直接构造一个容量为初始容量的 Object 数组，然后将其赋值给 elementData；
- 否则，当初始的容量等于 0 的时候，将 elementData 赋值为 EMPTY_ELEMENTDATA；
- 如果是负数，抛异常。
#### 1.3 基于其他 List 构造
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
- 首先通过 Collction 接口中定义的 toArray() 方法，将集合类转为一个数组，方便后续的处理；
- 然后在 if 的条件中去指定 size（当前 ArrayList 实例中存放元素的个数）为数组的长度，然后判断这个长度是否为 0；
- 如果 c.getClass() 为 ArrayList 的话，就直接将数组 a 赋值给 elementData ，如果不是的话，需要再做一次转化，通过 Arrays.copyOf() 方法将数组复制为对象数组，这样做的原因有两个：
	- 一是为了保证类型安全；
	- 二是为了防止那个自定义的集合将把自己内部的数组的引用传过来，此时两个 List 底层操控的是相同的数组，会引起很多意料之外的问题。
### 2 底层扩容机制
#### 2.1 判断是否需要扩容
```java
public boolean add(E e) {
	ensureCapacityInternal(size + 1);  // Increments modCount!!
	elementData[size++] = e;
	return true;
}
```
- 调用 ensureCapacityInternal() 来确保存储空间（elementData）足够放下这个新的元素，然后再将元素放入其中

```java
private void ensureCapacityInternal(int minCapacity) {
	ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

private static int calculateCapacity(Object[] elementData, int minCapacity) {
	if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
		return Math.max(DEFAULT_CAPACITY, minCapacity);
	}
	return minCapacity;
}


private void ensureExplicitCapacity(int minCapacity) {
	modCount++;
	// overflow-conscious code
	if (minCapacity - elementData.length > 0) grow(minCapacity);
}
```
- 使用 `calculateCapacity(elementData, minCapacity)` 方法去计算容量；
- 如果用户创建的时候没有指定初始容量，取 minCapacity 和 DEFAULT_CAPACITY 的最大值作为容量。
- 然后去判断 elementData 数组长度是否足够，如果不足就进入扩容方法 grow()。
#### 2.2 扩容方法
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
- 将 newCapacity 赋值为 oldCapacity 的 1.5 向下取整；
- 看能否满足 minCapacity，如果不能则本次扩容为 minCapacity
- 调用 Arrays.copyOf(elementData, newCapacity); 将原本 elementData 中的内容移动到新拓展的长度为 newCapacity 的数组中，这就完成了一个完整的扩容；
#### 2.3 空参构造与初始容量指定为 0
对于无参构造和初始容量为 0 的有参数构造，扩容的路径是不同的：
- 如果是无参构造，它带进来的 minCapacity 就是 10，最终其会被拓展为 10
- 如果是有参构造的话，带进来的 minCapacity 其实就是 1，且计算得 int newCapacity = oldCapacity + (oldCapacity >> 1); 结果是 0，那最终 elementData 会被拓展成 1，其扩容过程是这样的：

|minCapacity|newCapacity|最终值|
|---|---|---|
|1|0|1|
|2|1.5（1）|3|
|3|无需扩容|3|
|4|4.5（4）|4|
|5|6|6|
|6|无需扩容|6|
|7|9|9|
