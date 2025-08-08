## 1 Why G1?

### 1.1 CMS + ParNew 的核心痛点

CMS 是关注回收时间的收集器，在回收时使用标记清除算法，仅在无连续内存空间分配的时候才会进行整理和压缩，此时 CMS 的收集方式会退化成 Full GC，这是一个完全的 STW 操作，会暂停用户线程。

### 1.2 为什么会诞生 G1？

最大的原因是 JVM 截止当时还没有提供 **可控的 GC 回收时间** 的功能；

基于这样的考虑，早在 06 年就出现了 G1 的论文，但最终实装是在 Java1.7（2011 年）；这个收集器内部的结构注定是非常复杂的，当然我们没有必要去深入研究底层，只需要他的基础原理即可。

## 2 Core concept

在 G1 实现的过程中，引入了一些新的概念，让我们从这些概念开始来认识 G1。

### 2.1 Region

对于堆空间，传统的 GC 将这块区域划分成了新生代、老年代和永久代（Java8 移除，引入 Metaspace），这种划分方式的特点是各个分代的存储地址在逻辑上是连续的。

![[传统的 GC 布局w.png]]


而 G1 中各个分代的地址不是连续的，每种分代由一个个属于这个分代的区域（Region）构成；Region 是 G1 内存管理的最小单位，每个 Region 占用一块连续的内存地址：

![[G1 内存分配.png|500]]


除了上面标注的三种区域，G1 的堆中还有一个专门存放大对象的区域（Humongous）；G1 对于大对象的评判标准是大小大于等于 Region 一半的对象。

对于大对象，G1 不会在普通的区域中为它们分配内存。它会专门为大对象分配一个或多个**连续的 H 区域 (Humongous Regions)**，并且会有以下几个特殊处理：

- 大对象会被直接分配到老年代，防止其被反复的拷贝和移动；
- 未被引用的大对象会在全局并发标记的 clean-up 阶段和 full GC 阶段被回收；
- 在分配 H-obj 之前先检查是否超过 `-XX:InitiatingHeapOccupancyPercent=n` 的值，如果超过的话，就启动全局并发标记，为的是提早回收，防止 evacuation failures。

为了减小大对象的影响，可以调大 Region 的值，来将大对象变为普通对象；

Region 的大小可以通过参数 `-XX:G1HeapRegionSize` 设定，取值范围从 1M 到 32M，且是2 的指数。如果不设定，那么 G1 会根据 Heap 大小自动决定，Region 的值会被设定为：

$$\text{区域大小} = \max\left(\frac{\text{初始堆大小} + \text{最大堆大小}}{2 \times \text{目标区域数量}}, \text{最小区域大小}\right)$$

- 最小区域大小为 1M；
- (初始堆大小 + 最大堆大小) / 2 计算的是平均堆大小，用于平衡性能。

### 2.2 SATB 原始快照

STAB 原始快照，全称为 Snapshot-At-The-Beginning；它的作用是保证并发条件下 GC 标记的正确性；

为了提高 GC 的性能，其中最耗时的标记阶段一般会选择和用户线程并发执行；而标记阶段判断对象存活的方式是通过 GC Roots 的引用，而如果用户线程在执行标记阶段对部分引用进行了修改，可能会导致本不该回收的对象被错误的回收。

![[三色标记法.png|500]]

我们一般使用 “三色标记法” 来描述标记的过程，也就是将在扫描范围内的节点分为三种状态：白色，未被分析过的对象、灰色，正在被分析的对象、黑色，已经分析完成的对象；可以将整个扫描的过程看作白色对象逐步转化为黑色对象的过程。

而在这种情况下，本应该存活的对象会被回收：灰色节点删除了对这个对象的引用，而同时黑色节点新增了对这个节点的引用。

STAB 会在一个对象的引用被替换时，可以通过写屏障将旧引用记录下来，在结束后重新扫描这些被标记的对象；这样就类似在一个快照上进行对象的标记，在扫描过程中只关心引用在标记开始的这一瞬间的状态。

而对于新创建的对象，且灰色节点未持有对该对象的引用，上面的方法是无法处理的，因此，在 Region 中有两个 top-at-mark-start（TAMS）指针，分别为 `prevTAMS` 和 `nextTAMS`。在 TAMS 以上的对象是新分配的，这是一种隐式的标记。

### 2.3 RSet 记忆集

![[卡表和卡页的关系.png|400]]

在程序运行的过程中，不可避免的会出现一些跨代引用，比如说老年代对象中可能会存在堆新生代对象的引用，而在进行 Young GC 的时候，为了这一小部分引用（分代假说三中提到，跨代引用的数量很少），我们必须去扫描整个老年代；

为了解决这个问题，HotSpot 引入了卡表 Card Table 结构；

卡表会将整个老年代划分为一系列带下固定的卡片，每个卡片表示一块连续的内存区域。当一个老年代对象引用了新生代对象的时候，JVM 会通过写屏障来标记当前这块区域中存在对新生代对象的引用，这个操作称为 “变脏”。

这样，在进行扫描的时候，只需要扫描这些标记为脏的卡片，而不需要扫描整个老年代了，具体来说，是会将变脏的卡片区域的对象作为额外的的 GC Root。

---

G1 将堆区域划分成了一个个 Region，需要解决的问题从跨代引用变成了跨 Region 引用；

而 G1 采用了一个基于卡表思想的，却更为复杂的结构来存储跨 Region 引用，这个结构是 RSet，全称 Remembered Set，存储的是有哪些 Region 存在对当前 Region 的引用。

RSet 在 G1 中通过 post-write barrier 来维护。

RSet 是一个哈希表，Key 是别的 Region 的起始地址，Value 是一个集合，里面的元素是 Card Table 的 Index。

![[RSet、Card和Region的关系.png|500]]

比如如上图中，Region1 和 Region3 中存在对 Region2 内部对象的引用，那在 Region2 的 RSet 中，就会存储关于 Region1 和 Region3 的信息（例如它们的 Region ID），以及在 Region1 和 Region3 内部，那些包含对 Region2 引用对象的卡片索引。

### 2.4 Pause Prediction Model 停顿预测模型

G1 在回收时，使用暂停预测模型来尝试满足用户自定义的暂停时间目标，并根据指定的暂停时间目标选择要收集的区域数量。

暂停时间的目标通过 `-XX:MaxGCPauseMillis` 来指定，默认值为 200ms。

停顿预测模型是以衰减标准偏差为理论基础实现的：

```cpp
//  share/vm/gc_implementation/g1/g1CollectorPolicy.hpp
double get_new_prediction(TruncatedSeq* seq) {
    return MAX2(seq->davg() + sigma() * seq->dsd(),
                seq->davg() * confidence_factor(seq->num()));
}
```

在这个预测计算公式中：`davg` 表示衰减均值，`sigma()` 返回一个系数，表示信赖度，dsd表示衰减标准偏差，`confidence_factor` 表示可信度相关系数。而方法的参数 `TruncateSeq`，顾名思义，是一个截断的序列，它只跟踪了序列中的最新的 n 个元素。

在 G1 GC 过程中，每个可测量的步骤花费的时间都会记录到 `TruncateSeq`（继承了 `AbsSeq`）中，用来计算衰减均值、衰减变量，衰减标准偏差等：

```cpp
// src/share/vm/utilities/numberSeq.cpp

void AbsSeq::add(double val) {
  if (_num == 0) {
    // if the sequence is empty, the davg is the same as the value
    _davg = val;
    // and the variance is 0
    _dvariance = 0.0;
  } else {
    // otherwise, calculate both
    _davg = (1.0 - _alpha) * val + _alpha * _davg;
    double diff = val - _davg;
    _dvariance = (1.0 - _alpha) * diff * diff + _alpha * _dvariance;
  }
}
```

比如要预测一次 GC 过程中，RSet 的更新时间，每个 Dirty Card 的时间花费通过 `_cost_per_card_ms_seq` 来记录，具体预测代码如下：

```java
//  share/vm/gc_implementation/g1/g1CollectorPolicy.hpp

 double predict_rs_update_time_ms(size_t pending_cards) {
    return (double) pending_cards * predict_cost_per_card_ms();
 }
 double predict_cost_per_card_ms() {
    return get_new_prediction(_cost_per_card_ms_seq);
 }
```

## 3 GC Process

### 3.1 G1 GC 模式

G1 提供了两种 GC 模式，Young GC 和 Mixed GC，两种都是完全 Stop The World 的。 

- Young GC：选定所有年轻代里的 Region。通过控制年轻代的 region 个数，即年轻代内存大小，来控制 Young GC 的时间开销。 
- Mixed GC：选定所有年轻代里的 Region，外加根据 global concurrent marking 统计得出收集收益高的若干老年代 Region。在用户指定的开销目标范围内尽可能选择收益高的老年代 Region。

由上面的描述可知，Mixed GC 不是 full GC，它只能回收部分老年代的 Region；

如果 Mixed GC 的速度跟不上程序内存分配的速度，就会导致 Evacutaion Failed，进而触发 Serial Old GC，也就是 Full GC。

### 3.2 Young GC

当 Eden 区（年轻代的一部分）满时，G1 会暂停所有应用程序线程（STW）。

1. **确定回收范围 (CSet)：** G1 确定本次要回收的所有 Eden 区和当前所有 Survivor 区。
2. **根扫描：** 扫描 GC Roots（包括 RSet 中记录的老年代对年轻代的引用），找出所有存活的年轻代对象。
3. **复制存活对象：** 将这些存活的年轻代对象复制到新的 Survivor 区或直接晋升到老年代区。
4. **清空旧区域：** 原本的 Eden 区和旧的 Survivor 区被清空，变为可用的空闲 Region。
5. **结束：** 恢复应用程序线程。

### 3.3 并发标记

当堆占用率达到阈值、大对象分配导致老年代空间不足、动态 IHOP 的情况下，会触发并发标记阶段。

1. **初始标记**，STW；快速标记从 **GC Roots** (如线程栈变量、静态变量等程序当前正在使用的对象) 直接可达的少量对象，这些对象是后续并发标记的起点。 这个阶段的暂停时间通常非常短，因为它只进行浅层扫描，不进行深度遍历。    
2. **根区域扫描**，并发执行；扫描在 **初始标记** 阶段被标记出来的年轻代区域中仍然存活的对象所引用的 **老年代对象**。    
3. **并发标记**，并发执行；从 `Initial Mark` 和 `Root Region Scanning` 找到的存活对象开始，遍历整个堆（包括老年代和 H 区域），找出并标记所有可达（即存活）的对象。    
4. **重新标记**，STW；处理在 `Concurrent Mark` 阶段由于写屏障（SATB）记录下来的所有引用变化，确保没有存活对象被漏标。 
5. **清理**，并发执行；
	- 统计每个区域的存活度；
	- 识别并回收完全空闲的区域，如果一个 Region（老年代 Region + H-obj Region）在标记后发现完全没有存活对象，G1 会在这个阶段立即将其回收；
	- 为 Mixed GC 选择目标：G1 会根据存活度信息对老年代 Region 进行排序，识别出哪些 Region 的垃圾最多、回收效率最高，为后续的 **Mixed GC** 选择要回收的区域做好准备。

初始标记阶段共用了 Young GC 的暂停，这是因为他们可以复用 root scan 操作，所以可以说 global concurrent marking 是伴随 Young GC 而发生的；

清理阶段只是回收了没有存活对象的 Region，所以它并不需要 STW。

### 3.4 Mixed GC

Mixed GC 的由以下的几个参数控制：

| 参数名                               | 参数介绍                                                                                                                                              |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| `G1HeapWastePercent`              | G1 会在全局并发标记结束后，以及每一轮混合回收 (Mixed GC) 完成后，评估当前堆中可回收的垃圾总量。如果这些可回收的垃圾占堆总空间的比例低于 `G1HeapWastePercent` 设定的百分比，G1 就会认为继续进行混合回收的价值不大，从而停止当前的 Mixed GC 循环。 |
| `G1MixedGCLiveThresholdPercent`   | 老年代对象存活占比，只有在这个参数之下才会选入清理 Set 中。                                                                                                                  |
| `G1MixedGCCountTarget`            | 一次并发扫描后最多执行的 Mixed GC 次数。                                                                                                                         |
| `G1OldCSetRegionThresholdPercent` | 一次 Mixed GC 后能被选入 CSet 的最多的老年代 Region 个数。                                                                                                         |
1. **CSet 确定**：G1 将所有当前的年轻代区域 (Eden 和 Survivor) 加入回收集合；同时，根据之前并发标记的结果和预设的暂停时间目标，从垃圾最多的老年代区域中，选择**一部分**加入到本次的回收集合 (CSet) 中。    
2. **根扫描**：扫描所有常规的 GC Roots (如栈变量、静态变量等)。还会扫描 CSet 中所有 Region 的 Remembered Set (RSet)，RSet 中记录的来自外部的引用，被视为额外的 GC Roots，以确保被外部引用的对象不会被错误回收。   
3. **对象复制 (Evacuation)**：将所有在 CSet 内被标记为**存活**的对象（包括年轻代和老年代对象），复制到堆中**新的空闲 Region**。年轻代对象可能进入新的 Survivor 区或晋升老年代；老年代对象则复制到新的老年代区。这个复制过程也完成了内存碎片整理。    
4. **RSet 更新与维护**：如果一个对象被复制后，其引用的位置发生变化，G1 会更新目标 Region 的 RSet，以反映新的引用来源。同时，本次回收的 CSet 中所有被清空的旧 Region，其 RSet 信息也会被清理。
5. **引用处理**：处理各种特殊引用，如 弱引用、软引用、虚引用等。这些引用指向的对象可能会根据规则被回收。
6. **区域清空**：一旦所有存活对象都已复制走，CSet 中原来的所有 Region 都会被标记为完全空闲，并归还给 G1 的空闲 Region 列表，可供后续使用。

## 4 Q&A

### 4.1 G1 适合怎样的系统？

​G1适合的系统包括**超大内存**的系统，此时如果按照传统的分代，比如16G的内存新生代8G，老年代8G这种划分，虽然拉长垃圾停顿的时间，但是一旦新生代被占满，回收的效率是非常低的，因为对象和GC ROOT非常多，最终导致垃圾收集长时间卡顿。而G1不一样，他只需要按照算法判断根据停顿模型的值在新生代接近停顿模式时间的时候就马上开启回收，不用等新生代满了才回收。

​G1也适合**需要低延迟**的系统，因为低延迟对于系统的响应要求是非常高的，更加看重响应的时间，同时对于系统的资源要求也比较高，而分代的模型和理论在大内存机器会造成长时间的垃圾收集停顿，这对于实时响应的服务要求是非常高的。

### 4.2 G1 的缺点

1. 每个 Region 要更大的内存空间处理记忆集的消耗问题，还需要借助 TAMS 来辅助实现快照的功能。
2. cms使用写后屏障，而G1不仅使用写后屏障还用了写前屏障。额外使用队列进行异步并发标记的对象指针改动的问题（最终标记的时间开销）。



