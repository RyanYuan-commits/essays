---
title: Java 25 新特性 | Baeldung
source: https://www.baeldung.com/java-25-features
created: 2025-09-17
description: Java 25 作为一个新的 LTS 版本, 通过一系列语言、API 和运行时增强, 提升了开发体验、并发性能、安全性、内存效率和诊断能力, 为现代化应用开发提供了强大支持。
finished: "false"
tag: clipper
---
```embed
title: "GitHub - SimonVerhoeven/java25-demo"
image: "https://opengraph.githubassets.com/f01fecc03efdf935bf153f29afa13ad472d3c520741db0494e1b0b5e6a3367e7/SimonVerhoeven/java25-demo"
description: "Contribute to SimonVerhoeven/java25-demo development by creating an account on GitHub."
url: "https://github.com/SimonVerhoeven/java25-demo"
favicon: ""
aspectRatio: "50"
```

```embed
title: "New Features in Java 25 | Baeldung"
image: "https://www.baeldung.com/wp-content/uploads/2024/07/Java-Featured-10.jpg"
description: "Explore all the new features and changes introduced in Java 25."
url: "https://www.baeldung.com/java-25-features"
favicon: ""
aspectRatio: "52.33333333333333"
```


作为一名 Java 开发者, 如果您主要从事 Web 开发, 那么文档中提及的一些底层、并发和安全相关特性可能超出您日常工作范围, 建议您补充学习以下知识:

*   **JVM 内部原理与性能调优:** 
    *   **垃圾回收机制 (GC):** 特别是 Shenandoah 等现代 GC 的工作原理及其分代支持 (Generational Shenandoah, JEP 521)。
    *   **Java Flight Recorder (JFR):** 深入了解 JFR 如何进行 CPU 时间分析 (JEP 509)、协作采样 (JEP 518) 和方法计时与跟踪 (JEP 520), 以及如何解读其输出以进行性能诊断。
    *   **对象内存布局:** 理解 Compact Object Headers (JEP 519) 如何影响内存占用。
    *   **Ahead-of-Time (AOT) 编译:** 了解 AOT 的概念及其在 Java 25 中的初步支持 (JEP 514, JEP 515), 以及它如何影响启动性能和内存占用。
*   **并发编程进阶:** 
    *   **Scoped Values (JEP 506):** 理解其作为 `ThreadLocal` 替代品在虚拟线程 (Virtual Threads) 环境下的使用和优势。
    *   **Structured Concurrency (JEP 505):** 掌握结构化并发模型, 如何管理相关联的并发任务组。
*   **密码学基础与安全 API:** 
    *   **PEM 编码:** 了解 PEM (Privacy-Enhanced Mail) 格式在加密对象 (如密钥、证书) 中的作用, 以及 Java 25 对其的标准化支持 (JEP 470)。
    *   **Key Derivation Function (KDF):** 学习密码派生函数 (如 PBKDF2, scrypt) 的基本原理及其在 Java 25 中的标准化 API (JEP 510) 以增强密码存储安全性。
*   **低级硬件优化:** 
    *   **Vector API (JEP 508):** 了解 SIMD (Single Instruction, Multiple Data) 指令集, 以及 Vector API 如何在 Java 中实现数据并行计算以提升性能。

**关键词:**
Java 25, LTS, 语言特性, API增强, JEP, Pattern Matching, Module Import, Compact Source Files, Flexible Constructors, Scoped Values, Structured Concurrency, Stable Value, Cryptography, PEM, KDF, Vector API, JFR, AOT, GC, Shenandoah, 性能, 并发, 预览特性。

**详细总结:**
Java 25, 作为下一个长期支持 (LTS) 版本, 于2025年9月发布, 引入了广泛的语言、API 和运行时增强。在语言和编译器层面, 它通过 **JEP 507** 允许在模式匹配中处理原始类型, 引入 **JEP 511** 的模块导入声明 (`import module`) 以改善模块依赖的可读性, **JEP 512** 支持紧凑源文件和实例 `main` 方法以简化脚本编写, 并通过 **JEP 513** 实现灵活的构造器体, 允许在 `super()` 或 `this()` 调用前执行代码。

API 方面, **JEP 506** 提供了 Scoped Values 作为 `ThreadLocal` 的轻量级、不可变、线程安全替代品, 特别适用于虚拟线程; **JEP 505** 进一步完善了结构化并发 API, 简化了并发任务的管理; **JEP 502** 引入了 Stable Value API, 提供类似 `Optional` 的上下文稳定不可变值; 在安全方面, **JEP 470** 标准化了 PEM 格式加密对象的读写支持, **JEP 510** 则提供了密码派生函数 (KDF) 的标准 API。此外, **JEP 508** 的 Vector API 作为第十次孵化器提案, 持续为数据并行计算提供高效的硬件指令支持。

其他重要变更包括: **JEP 503** 移除了对 32 位 x86 端口的支持; **JEP 509、518、520** 对 JFR (Java Flight Recorder) 进行了多项增强, 包括 CPU 时间分析、协作采样和方法计时与跟踪, 大幅提升了性能诊断能力; **JEP 514 和 515** 为 AOT (Ahead-of-Time) 编译奠定了基础, 旨在改善启动性能和内存占用; **JEP 519** 引入了紧凑对象头以减少内存消耗; 而 **JEP 521** 为 Shenandoah 垃圾回收器增加了分代支持, 进一步优化了吞吐量和暂停时间。文档最后提醒开发者, 许多特性仍处于预览或孵化阶段, 使用时需通过特定命令行参数启用, 并在生产环境中谨慎使用。