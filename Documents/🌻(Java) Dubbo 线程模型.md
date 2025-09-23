---
type: Java
sub-type: Dubbo
finished: "false"
---
## 1 消费端线程模型

在 Dubbo 2.7.5 之前的消费端 Dubbo 应用, 在面对大量服务且并发比较大的大流量场景时, 经常会出现 Consumer 端线程数分配过多的问题.

```embed
title: "Need a limited Threadpool in consumer side · Issue #2013 · apache/dubbo"
image: "https://opengraph.githubassets.com/23c7e58e26482e5b1cdffc1d32ad083be590b3b4b1457e875f78c36d78d4fce2/apache/dubbo/issues/2013"
description: "Environment Dubbo version: 2.5.3 Java version: 1.7 in some circumstances, too many threads will be created and thus process will suffer from OOM.Here is the related issue #1932 After analyzing the ..."
url: "https://github.com/apache/dubbo/issues/2013"
favicon: ""
aspectRatio: "50"
```

老的线程池模型

![[Dubbo 线程模型-old.png|600]]

问题在于, 每个 NettyClient 都有自己私有的线程池, 而且这些线程池的线程数量是没有上限的.






























---

参考内容

- Dubbo 官方文档: https://cn.dubbo.apache.org/zh-cn/overview/mannual/java-sdk/tasks/framework/threading-model/
- issue #2023 回复 by HelloLyfing: https://cn.dubbo.apache.org/zh-cn/overview/mannual/java-sdk/tasks/framework/threading-model/