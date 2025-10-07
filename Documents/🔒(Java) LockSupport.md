---
type: Java
sub-type: 并发编程
created: 2025-10-02 23:01:01
updated: 2025-10-03 11:14:57
---
`LockSupoort` 是一个提供线程阻塞和唤醒功能的工具类, Java 各种同步组件底层的线程调度能力是 `LockSupport` 提供的.

## 1 核心价值

`LockSupport` 的设计初衷是为了客服 `Object.wait/notify` 机制的一些固有缺点:

| 特性        | `Object.wait/notify`                                   | `LockSupport.park/unpark`       | **优势解读**                      |
| --------- | ------------------------------------------------------ | ------------------------------- | ----------------------------- |
| **与锁的关系** | 必须在 `synchronized` 同步块内调用                              | 无需获取任何锁, 在任意位置调用                | 更灵活, 不用和 `synchronized` 绑定    |
| **唤醒时序**  | 必须先 `wait` 再 `nofify`, 如果 `notify` 先发生, 信号会丢失, 线程将永久等待 | `unpark` 可以先于 `park` 发生, 信号不会丢失 | 更安全, 彻底解决了线程间因时序竞态导致的信号修饰的问题. |
| **唤醒精度**  | `notify` 随机唤醒一个等待线程, `notifyAll` 唤醒所有                  | `unpark(Thread t)` 精确唤醒指定的咸亨    | 更精确, 避免了不必要的上下文切换和 "惊群效应".    |

## 2 核心原理: 许可模型

每个线程都有一个关联的 "许可", 这个许可最多只能有一个 (是 0 或者 1 的状态).

当调用 `park` 方法的时候, 如果当前线程的 Permit 是 1, 则消耗掉 Permit 并**立刻返回**, 而如果 Permit 为 0, 则线程阻塞, 直到 Permit 变为 1.

当调用 `unpark` 时, 会将 Permit 变为 1:

- 如果线程之前因为 `park` 被阻塞, 则其将被唤醒;
- 如果 `t` 尚未 `park`, 则它下次调用 `park` 时会直接通过.

## 3 核心 API

`static void park()` 如果 Permit 为 0, 阻塞当前线程;

`static void unpark(Thread thread)`: 唤醒当前线程, 将 Permit 置为 1;

`static void parkNanos(long nanos)`: 带超时的阻塞;

`static void parkUtil(long deadline)`: 阻塞到某个时间点.

## 4 使用案例

```java
Thread mainThread = Thread.currentThread();  
  
Thread thread = new Thread(() -> {  
    try {  
        Thread.sleep(2000);  
    } catch (InterruptedException e) {  
        throw new RuntimeException(e);  
    }  
    LockSupport.unpark(mainThread);  
});  
  
thread.start();  
System.out.println("主线程: 等待子线程完成");  
LockSupport.park();  
System.out.println("主线程: 已被唤醒, 程序结束");
```

