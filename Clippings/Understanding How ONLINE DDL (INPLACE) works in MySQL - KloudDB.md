---
title: "理解 MySQL 中 ONLINE DDL (INPLACE) 的工作原理 - KloudDB"
source: "https://klouddb.io/understanding-how-online-ddl-inplace-works-in-mysql/"
created: 2025-10-22
description: "本文详细解析了 MySQL INPLACE 在线 DDL 的执行机制，包括完整的 12 步流程、锁管理、并行 DML 处理，以及 MySQL 5.7 与 8.0 版本的差异。"
finished: "false"
tag: "clipper"
---
需要补充学习的知识：
- MySQL 数据字典与存储引擎架构
- MySQL DDL 锁机制（MDL 锁）
- InnoDB 索引结构与 B+树构建算法
- 数据库事务与并发控制

关键词：
MySQL, ONLINE DDL, INPLACE, 锁机制, 数据字典, 表重建

详细总结：
本文深入解析了 MySQL 中 INPLACE（在线 DDL）的工作原理，对比了 COPY、INPLACE 和 INSTANT 三种 DDL 执行方式的差异。重点阐述了 INPLACE DDL 的完整执行流程，包括 12 个关键步骤：从解析 ALTER 语句、获取锁、创建中间表，到准备阶段、数据复制阶段、应用并行 DML 变更，最后提交变更。文章详细说明了 INPLACE 的三种变体（仅更新元数据、添加新对象、表重建），并解释了 MySQL 5.7 与 8.0 在数据字典统一性和 DDL 原子性方面的改进。同时解答了关于并行 DML 处理、锁机制、崩溃恢复等常见问题。