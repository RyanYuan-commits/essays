---
type: Java
sub-type: JVM
---
ZGC 是一个 low latency 的 Java 垃圾收集器, 设计用来解决大堆的垃圾收集问题, ZGC is highly scalable , 支持 8MB 到 16 TB 空间的垃圾收集; 它使用 Concurrent marking 和 Concurrent relocation 技术来实现极低的延迟, rarely exceed 250 microseconds;

## 1 回收流程

ZGC 的垃圾回收过程包含一下几个阶段:

- **标记开始 Pause Mark Start:** 一个短暂的 STW 停顿, 仅用于标记 GC Roots 指向的初始对象集；
- **并发标记 Concurrent Mark:** 与应用程序并行执行, 遍历对象图并标记所有存活的对象；
- **标记结束 Pause Mark End:** 一个短暂的 STW 停顿, 用于处理并发标记期间的同步和边界情况, 其耗时与堆大小无关；
- **并发预备重定位 Concurrent Prepare for Relocate:** 识别出需要被压缩和移动的区域 (Relocation Set), 并为下一阶段做准备；
- **重定位开始 Pause Relocate Start:** 一个短暂的 STW 停顿, 用于重定位 GC Roots 引用的对象, 并确保所有线程都准备好进入并发迁移；
- **并发重定位 Concurrent Relocate:** 核心的迁移阶段, 将存活对象复制到新的内存区域, 并通过读屏障 (Load Barrier) 机制确保应用线程能正确访问对象的新地址. 

## 2 内存分布

ZGC 与传统的垃圾收集器不同, 没有分代的概念, 而是采用了类似 G1 的 Region 机制, ZGC 将堆内存划分为三个不同规格的区域:

- Small Region: 大小为 2 MB, 存放小于 256 KB 的对象;
- Medium Region: 大小为 32 MB, 用于存放 256 KB ~ 4 MB 之间的对象;
- Large Region: 容量可动态调整 (2 MB 的倍数), 用于存放大于 4MB 的对象, 这块区域的对象不会迁移, 因为复制大对象的开销极其高昂.

## 3 触发机制

ZGC 核心特性之一是并发性, 需要确保在垃圾回收完成之前, 堆不会被塞满, 其主要触发机制包括:

- **Blocking Memory Allocation Requests**: 当堆内存填充速度超过垃圾回收能力时, 线程可能会被阻塞, 日志关键字为 "Allocation Stall";
- **Adaptive Algorithm Based on Allocation Rate**: 根据近期的分配速率以及 GC 次数动态计算何时触发垃圾回收, 日志关键字为 "Allocation Rate";
- **Fixed Time Intervals**: 可以基于固定时间间隔触发, 在应对突发流量场景下非常有用, 日志关键字为 "Timer";
- **Warmup Phase**: 发生在服务启动期间, 无需特别关注, 日志关键字为 "Warmup";
- **External Trigger**: 显示的调用 `System.gc()`, 日志关键字为 "System.gc()".

## 4 Key Innovations in ZGC

### 4.1 Colored Pointers

![[ZGC colored pointers.png|700]]

上面展示的是一个 ZGC 指针, 开头的 18 位未被使用, 最后的 42 位用于地址定位, 可以定位 4TB 的地址; Marked0 和 Marked1 用于表示对象在当前 GC 周期中是否被标记为存活, 使用两个标志位来避免在下个周期开始之前对标志位的清理工作; Remapped 用于表示改对象是否已经被 Relocation 到新的地址;  Finallizable 用于标记这个对象无法正常使用, 仅可被 finalizer 访问.

### 4.2 Load Barriers

ZGC 通过使用加载屏障来保证了对象访问期间的内存一致性;

加载屏障根据指针颜色, 来确保对象在垃圾回收过程中被移动时, 指针能够更新到正确的位置.

### 4.3 Memory Multi-Mapping

ZGC 将同一个物理内存映射到多个虚拟地址, 每个虚拟地址代表不同的垃圾回收状态, 这使得 ZGC 可以在不同的内存视图之间切换, 而无需实际移动数据.

## 附录

### ZGC 参数

| **JVM 参数**                                       | **说明**                            |
| ------------------------------------------------ | --------------------------------- |
| `-Xms -Xmx`                                      | 将最小和最大堆大小都设置为 10GB                |
| `-XX:ReservedCodeCacheSize`                      | 设置用于 JIT 编译代码的代码缓存大小              |
| `-XX:+UnlockExperimentalVMOptions -XX:+UseZGC`   | 在 JVM 中启用 ZGC                     |
| `-XX:ConcGCThreads`                              | 指定并发垃圾回收的线程数量, 默认为 CPU 核心数的 12.5% |
| `-XX:ParallelGCThreads`                          | 定义 STW 阶段的线程数量。默认为 CPU 核心数的 60%   |
| `-XX:ZCollectionInterval`                        | ZGC 垃圾回收之间的最小时间间隔, 以**秒**为单位      |
| `-XX:ZAllocationSpikeTolerance`                  | 根据分配峰值调整 ZGC 的触发时机                |
| `-XX:+UnlockDiagnosticVMOptions -XX:-ZProactive` | 控制主动垃圾回收行为                        |
| `-Xlog`                                          | 配置垃圾回收事件的日志记录                     |

### 其他资源

```embed
title: "ZGC, the JDK's Newest Garbage Collector - Sip of Java – Inside.java"
image: "https://inside.java/images/java-cup.png"
description: "Let's explore the JDK's newest garbage collector, ZGC."
url: "https://inside.java/2022/05/30/sip053/"
favicon: ""
aspectRatio: "93.75"
```
