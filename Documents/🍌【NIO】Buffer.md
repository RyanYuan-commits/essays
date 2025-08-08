# 基本介绍
应用程序与通道（Channel）主要的交互，主要是进行数据的read读取和write写入。为了完成 NIO 的非阻塞读写操作，NIO为大家准备了第三个重要的组件——NIO Buffer（NIO缓冲区）。
Buffer 顾名思义：缓冲区，实际上是一个容器，一个连续数组。Channel 提供从文件、网络读取数据的渠道，但是读写的数据都必须经过 Buffer。
![[client 与 server 通过 Buffer 和 channel 连接.png#pic_center|1000]]
所谓通道的读取，就是将数据从通道读取到缓冲区中；所谓通道的写入，就是将数据从缓冲区中写入到通道中。
缓冲区的使用，是面向流进行读写操作的 OIO 所没有的，也是 NIO 非阻塞的重要前提和基础之一。
# Buffer 类及其属性
Buffer类是一个抽象类，对应于Java的主要数据类型，在NIO中有8种缓冲区类，分别如下：ByteBuffer、CharBuffer、DoubleBuffer、FloatBuffer、IntBuffer、LongBuffer、ShortBuffer、MappedByteBuffer。
前7种Buffer类型，覆盖了能在IO中传输的所有的Java基本数据类型。第8种类型 MappedByteBuffer 是专门用于内存映射的一种 ByteBuffer 类型。
不同的 Buffer 子类，其能操作的数据类型能够通过名称进行判断，比如IntBuffer只能操作Integer类型的对象。
## Buffer 类的重要属性
**Buffer 的子类会拥有一块内存，作为数据的读写缓冲区**，但是读写缓冲区并没有定义在 Buffer 基类，而是定义在具体的子类中。
如 ByteBuffer 子类就拥有一个 byte[] 类型的数组成员 final byte[] hb，作为自己的读写缓冲区，数组的元素类型与Buffer子类的操作类型相互对应。为了记录读写的状态和位置，Buffer 类额外提供了一些重要属性：capactiry（容量）、position（读写位置）、limit（读写的限制）、mark（读写位置的临时备份）。其中，position（读写位置）是读和写共用的标识，所以读写切换的时候需要手动调用方法来转化 position 的值，否则会出现错误。
### capacity 属性
Buffer类的capacity属性，表示内部容量的大小。一旦写入的对象数量超过了capacity容量，缓冲区就满了，不能再写入了。
**Buffer类的capacity属性一旦初始化，就不能再改变**。Buffer类的对象在初始化时，会按照capacity分配内部数组的内存，在数组内存分配好之后，它的大小当然就不能改变了。前面讲到，Buffer 类是一个抽象类，Java 不能直接用来新建对象。在具体使用的时候，必须使用 Buffer 的某个子类，例如 DoubleBuffer 子类，该子类能写入的数据类型是double类型，如果在创建实例时其 capacity 是 100，那么我们最多可以写入 100 个 double 类型的数据。
### position 属性
>[!info] 
>Buffer 类的 position 属性，表示当前的位置。position 属性的值与缓冲区的读写模式有关。
>在不同的模式下，position 属性值的含义是不同的，在缓冲区进行读写的模式改变时，position 值会进行相应的调整。

**在写入模式下，position 的值变化规则如下**：
	（1）在刚进入到写入模式时，position 值为 0，表示当前的写入位置为从头开始。
	（2）每当一个数据写到缓冲区之后，position 会向后移动到下一个可写的位置。
	（3）初始的 position 值为 0，最大可写值为 limit–1。当 position 值达到 limit 时，缓冲区就已经无空间可写了。
	
**在读模式下，position 的值变化规则如下**：
	（1）当缓冲区刚开始进入到读取模式时，position 会被重置为0。
	（2）当从缓冲区读取时，也是从 position 位置开始读。读取数据后，position 向前移动到下一个可读的位置。
	（3）在读模式下，limit 表示可以读上限。position 的最大值，为最大可读上限 limit，当 position 达到 limit 时，表明缓冲区已经无数据可读。

当新建了一个缓冲区实例时，缓冲区处于写入模式，这时是可以写数据的。在数据写入完成后，如果要从缓冲区读取数据，这就要进行模式的切换，可以调用 flip 翻转方法，将缓冲区变成读取模式。
**在从写入模式到读取模式的flip翻转过程中，position和limit属性值会进行调整，具体的规则是**：
	（1）limit 属性被设置成写入模式时的 position 值，表示可以读取的最大数据位置；
	（2）position 由原来的写入位置，变成新的可读位置，也就是 0，表示可以从头开始读。
### limit 属性
Buffer 类的 limit 属性，表示可以 **写入或者读取** 的最大上限，其具体的含义和读写模式有关。
	（1）在写入模式下，limit属性值的含义为可以写入的数据最大上限。在刚进入到写入模式时，limit的值会被设置成缓冲区的capacity容量值，表示可以一直将缓冲区的容量写满。
	（2）在读取模式下，limit的值含义为最多能从缓冲区中读取到多少数据。一般来说，在进行缓冲区操作时，是先写入然后再读取的。当缓冲区写入完成后，就可以开始从Buffer读取数据，可以使用flip翻转方法，这时，limit的值也会进行调整。具体如何整呢？将写入模式下的position值，设置成读取模式下的limit值，也就是说，将之前写入的最大数量，作为可以读取的上限值。
## Buffer 类的重要方法
### allocate() 创建缓冲区
演示如何获取一个整型的 Buffer 对象
```java
public class BufferSourceRead {  
  
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {  
        IntBuffer buffer = IntBuffer.allocate(20);  
  
        Field mark = Buffer.class.getDeclaredField("mark");  
        mark.setAccessible(true);  
  
        System.out.println("position = " + buffer.position());  
        System.out.println("limit = " + buffer.limit());  
        System.out.println("capacity = " + buffer.capacity());  
        System.out.println("mark = " + mark.get(buffer));  
    }  
  
}
```
通过调用 `IntBuffer allocate(20)`，创建了一个 IntBuffer 的实例对象，并分配了 $20 * 4$ 个字节。
其内部结构如下：
![[Buffer 初始化后各属性默认值.png#pic_center|800]]
上面代码输出的结果为：
```
position = 0
limit = 20
capacity = 20
mark = -1
```
从上面的运行结果，可以看出：一个缓冲区在新建后，处于写入的模式，position 属性（代表写入位置）的值为0，缓冲区的 capacity 容量值也是初始化时allocate方法的参数值（这里是20），而 limit 最大可写上限值也为的 allocate 方法的初始化参数值。
### put() 写入到缓冲区
```java
public class BufferSourceRead {  
  
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {  
        IntBuffer buffer = IntBuffer.allocate(20);  
  
        Field mark = Buffer.class.getDeclaredField("mark");  
        mark.setAccessible(true);  
  
        for (int i = 0; i < 5; i++) {  
            buffer.put(i);  
        }  
  
        System.out.println("position = " + buffer.position());  
        System.out.println("limit = " + buffer.limit());  
        System.out.println("capacity = " + buffer.capacity());  
        System.out.println("mark = " + mark.get(buffer));  
    }  
  
}
```
输出结果为：
```
position = 5
limit = 20
capacity = 20
mark = -1
```
其内部结构变为了这样：
![[put 写入后 Buffer 内部的状况.png#pic_center|800]]
### flip() 反转
```java
public class BufferSourceRead {  
  
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {  
        IntBuffer buffer = IntBuffer.allocate(20);  
  
        Field mark = Buffer.class.getDeclaredField("mark");  
        mark.setAccessible(true);  
  
        for (int i = 0; i < 5; i++) {  
            buffer.put(i);  
        }  
  
        buffer.flip();  
  
        System.out.println("position = " + buffer.position());  
        System.out.println("limit = " + buffer.limit());  
        System.out.println("capacity = " + buffer.capacity());  
        System.out.println("mark = " + mark.get(buffer));  
    }  
  
}
```
输出结果为：
```
position = 0
limit = 5
capacity = 20
mark = -1
```
其内部结果变为了这样：
![[flip 翻转后的 Buffer 内部结构.png#pic_center|800]]
flip 具体的源码是这样的：
```java
public final Buffer flip() {  
    limit = position;  
    position = 0;  
    mark = -1;  
    return this;  
}
```
具体步骤是这样的：
	（1）首先，设置可读上限limit的属性值。将写入模式下的缓冲区中内容的最后写入位置position值，作为读取模式下的limit上限值。
	（2）其次，把读的起始位置position的值设为0，表示从头开始读。
	（3）最后，清除之前的mark标记，因为mark保存的是写入模式下的临时位置，发生模式翻转后，如果继续使用旧的mark标记，会造成位置混乱。
### get() 从缓冲区读取数据
```java
public class BufferSourceRead {  
    private static final IntBuffer buffer = IntBuffer.allocate(10);  
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {  
        for (int i = 0; i < 5; i++) {  
            buffer.put(i);  
        }  
        buffer.flip();  
        // 先获取两个数据
        for (int i = 0; i < 2; i++) {  
            System.out.println(buffer.get());  
        }  
        log();  
        System.out.println("=========");  
        for (int i = 0; i < 3; i++) {  
            System.out.println(buffer.get());  
        }  
        log();  
    }  
    private static void log() throws NoSuchFieldException, IllegalAccessException {  
        Field mark = Buffer.class.getDeclaredField("mark");  
        mark.setAccessible(true);  
        System.out.println("position = " + buffer.position());  
        System.out.println("limit = " + buffer.limit());  
        System.out.println("capacity = " + buffer.capacity());  
        System.out.println("mark = " + mark.get(buffer));  
    }  
}
```
输出结果：
```
0
1
position = 2
limit = 5
capacity = 10
mark = -1
=========
2
3
4
position = 5
limit = 5
capacity = 10
mark = -1
```
从程序的输出结果，我们可以看到，读取操作会改变可读位置 position 的属性值，而 limit 可读上限值并不会改变。在position值和limit的值相等时，表示所有数据读取完成，position 指向了一个没有数据的元素位置，已经不能再读了。此时再读，会抛出 BufferUnderflowException 异常。
那么，在读完之后是否可以立即对缓冲区进行数据写入呢？答案是不能。现在还处于读取模式，我们必须调用 Buffer.clear()或Buffer.compact() 方法，即清空或者压缩缓冲区，将缓冲区切换成写入模式，让其重新可写。
此外还有一个问题：缓冲区是不是可以重复读呢？答案是可以的，既可以通过倒带方法 rewind() 去完成，也可以通过 mark() 和 reset() 两个方法组合实现。
### rewind() 倒带
我们在上一部分的代码上调用一下 rewind 方法
```java
public class BufferSourceRead {  
    private static final IntBuffer buffer = IntBuffer.allocate(10);  
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {  
        for (int i = 0; i < 5; i++) {  
            buffer.put(i);  
        }  
        buffer.flip();  
        for (int i = 0; i < 2; i++) {  
            buffer.get();  
        }  
        for (int i = 0; i < 3; i++) {  
            buffer.get();  
        }  
        buffer.rewind();  
        log();  
    }  
    private static void log() throws NoSuchFieldException, IllegalAccessException {  
		// 打印当前 Buffer 的内部情况
    }  
}
```
输出结果：
```
position = 0
limit = 5
capacity = 10
mark = -1
```

rewind 的原理上这样的：
```java
public final Buffer rewind() {  
    position = 0;  
    mark = -1;  
    return this;  
}
```
具体规则：
	（1）position重置为0，所以可以重读缓冲区中的所有数据；
	（2）limit保持不变，数据量还是一样的，仍然表示能从缓冲区中读取的元素数量；
	（3）mark标记被清理，表示之前的临时位置不能再用了。
### mark() 和 reset()
mark() 和 reset() 两个方法是成套使用的：
	Buffer.mark() 方法将当前 **position** 的值保存起来，放在 mark 属性中，让 mark 属性记住这个临时位置；之后，可以调用 Buffer.reset() 方法将mark的值恢复到 position 中。
mark 方法：
```java
public final Buffer mark() {  
    mark = position;  
    return this;  
}
```

reset 方法：
```java
public final Buffer reset() {  
    int m = mark;  
    if (m < 0)  
        throw new InvalidMarkException();  
    position = m;  
    return this;  
}
```
### clear() 清空缓冲区
```java
public final Buffer clear() {  
    position = 0;  
    limit = capacity;  
    mark = -1;  
    return this;  
}
```
重新回归到最初的状态，可以看到，缓冲区内部的数据并没有被清空，只是移动了指针。
## 使用 Buffer 类的基本步骤
总体来说，使用Java NIO Buffer类的基本步骤如下:
	（1）使用创建子类实例对象的allocate( )方法，创建一个Buffer类的实例对象。
	（2）调用put( )方法，将数据写入到缓冲区中。
	（3）写入完成后，在开始读取数据前，调用Buffer.flip( )方法，将缓冲区转换为读模式。
	（4）调用get( )方法，可以从缓冲区中读取数据。
	（5）读取完成后，调用Buffer.clear( )方法或Buffer.compact()方法，将缓冲区转换为写入模式，可以继续写
