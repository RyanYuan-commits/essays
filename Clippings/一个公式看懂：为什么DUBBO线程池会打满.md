---
title: 一个公式看懂：为什么DUBBO线程池会打满
source: https://www.cnblogs.com/ceshi2016/p/17286921.html
created: 2025-09-02
description: 本文通过“并发量 = QPS x RT”公式，深入剖析了DUBBO线程池打满的原因，主要归结为慢服务、预热不足导致的RT上升和流量激增导致的QPS上升，并提供了相应的排查和解决策略。
finished: "false"
tag: clipper
cover: https://cn.dubbo.apache.org/imgs/dubbo_colorful.png
---
