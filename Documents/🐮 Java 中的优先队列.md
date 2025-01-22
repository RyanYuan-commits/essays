`PriorityQueue` 是 Java 集合框架中的一个类，用于实现优先级队列。优先级队列是一种特殊的队列，其中每个元素都有一个优先级，出队时总是优先级最高的元素先出队。`PriorityQueue` 是基于堆（heap）数据结构实现的，默认情况下是一个最小堆。
### 特性
1. **自动排序**: `PriorityQueue` 会根据元素的自然顺序（或通过提供的比较器）自动对元素进行排序。
2. **非线程安全**: `PriorityQueue` 不是线程安全的。如果需要在多线程环境中使用，可以使用 `PriorityBlockingQueue`。
3. **不允许 `null` 元素**: `PriorityQueue` 不允许插入 `null` 元素。
4. **无界队列**: `PriorityQueue` 是一个无界队列，但其容量可以根据需要自动增加。
### 构造方法
`PriorityQueue` 提供了多种构造方法：
- `PriorityQueue()`: 创建一个默认初始容量为 11 的空优先级队列。
- `PriorityQueue(int initialCapacity)`: 创建一个具有指定初始容量的空优先级队列。
- `PriorityQueue(int initialCapacity, Comparator<? super E> comparator)`: 创建一个具有指定初始容量和比较器的优先级队列。
- `PriorityQueue(Collection<? extends E> c)`: 创建一个包含指定集合元素的优先级队列。
- `PriorityQueue(PriorityQueue<? extends E> c)`: 创建一个包含指定优先级队列元素的优先级队列。
- `PriorityQueue(SortedSet<? extends E> c)`: 创建一个包含指定排序集元素的优先级队列。
### 常用方法
- `boolean add(E e)`: 将指定元素插入到优先级队列中。
- `boolean offer(E e)`: 将指定元素插入到优先级队列中。
- `E poll()`: 获取并移除队列头部的元素，如果队列为空，则返回 `null`。
- `E remove()`: 获取并移除队列头部的元素。
- `E peek()`: 获取但不移除队列头部的元素，如果队列为空，则返回 `null`。
- `E element()`: 获取但不移除队列头部的元素。
- `boolean isEmpty()`: 检查队列是否为空。
- `int size()`: 返回队列中的元素数量。
### 示例代码
```java
import java.util.PriorityQueue;
import java.util.Collections;

public class PriorityQueueExample {
    public static void main(String[] args) {
        // 创建一个最小堆
        PriorityQueue<Integer> minHeap = new PriorityQueue<>();
        minHeap.add(10);
        minHeap.add(20);
        minHeap.add(15);

        System.out.println("Min Heap:");
        while (!minHeap.isEmpty()) {
            System.out.println(minHeap.poll()); // 输出按升序排列的元素
        }

        // 创建一个最大堆
        PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());
        maxHeap.add(10);
        maxHeap.add(20);
        maxHeap.add(15);

        System.out.println("Max Heap:");
        while (!maxHeap.isEmpty()) {
            System.out.println(maxHeap.poll()); // 输出按降序排列的元素
        }
    }
}
```
在这个示例中，我们创建了一个最小堆和一个最大堆，并演示了如何使用 `PriorityQueue` 来管理和访问这些元素。
