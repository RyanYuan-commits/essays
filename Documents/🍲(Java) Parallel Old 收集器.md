x## 1 Parallel Scavenge 的老年代版本
Parallel Old是 Parallel Scavenge 收集器的老年代版本，支持多线程并发收集，基于**标记-整理算法**实现。

这个收集器是直到 JDK 6 时才开始提供的，在此之前，新生代的 Parallel Scavenge 收集器一直处于相当尴尬的状态，原因是如果新生代选择了 Parallel Scavenge 收集器，老年代除了 Serial Old 收集器以外别无选择，其他表现良好的老年代收集器，如 CMS 无法与它配合工作。

由于老年代 Serial Old 收集器在服务端应用性能上的“拖累”，使用 Parallel Scavenge 收集器也未必能在整体上获得吞吐量最大化的效果。

同样，由于单线程的老年代收集中无法充分利用服务器多处理器的并行处理能力，在老年代内存空间很大而且硬件规格比较高级的运行环境中，这种组合的总吞吐量甚至不一 定比 ParNew 加 CMS 的组合来得优秀。直到Parallel Old 收集器出现后，“吞吐量优先” 收集器终于有了比较名副其实的搭配组合，在**注重吞吐量或者处理器资源较为稀缺的场合**，都可以优先考虑 Parallel Scavenge 加 Parallel Old 收集器这个组合。
