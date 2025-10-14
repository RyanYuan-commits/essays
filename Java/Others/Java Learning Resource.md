---
type: Java
sub-type: Resource
---
> 面对 Java 版本更新的复杂信息, 本文提供一份清晰的阅读指南, 帮你快速的理清各种文档的区别, 了解在在哪个文档中能获取到哪些知识.
## 1 JDK Release Notes

```embed
title: "JDK Release Notes | Oracle 中国"
image: "https://www.oracle.com/asset/web/i/flg-cn.svg"
description: ""
url: "https://www.oracle.com/cn/java/technologies/javase/jdk-relnotes-index.html"
favicon: ""
aspectRatio: "100"
```

涵盖内容: 新特性, 重要变更, 已知问题, 安全更新, 兼容性说明, 与其他版本的对照;

每个子版本, 如 JDK9, JDK9.0.1, JDK9.0.4 均有相关的 Release Notes, 颗粒度较细, 涵盖内容较多;

开发者可以重点关注新特性和重要变更部分, 了解如何使用新特性改进代码, 以及可以阅读

## 2 Java SE Documentation

```embed
title: "Java Platform, Standard Edition Documentation - Releases"
image: "https://docs.oracle.com/en/java/javase/sp_common/shared-images/1-java.png"
description: "Java Platform, Standard Edition documentation, current and previous releases"
url: "https://docs.oracle.com/en/java/javase/index.html"
favicon: ""
aspectRatio: "82.6086956521739"
```

是面向特定版本的, 百科全书式的官方技术规范合集. 

文档的内容跨版本累积, 每个新版本的文档都会在旧版基础上增删改, 以精确描述该版本 Java 平台的完整状态.

下面简单介绍下文档中值得关注的部分.
### 2.1 Tools Release

Java SE 工具参考, 介绍了 JDK 中各种工具和命令的使用方式; 

比如 javac, javap 等主要工具, jps, jmc 等监控工具...

### 2.2 Language Update

主要关注大版本在具体语法层面的变更变更, 比如 JDK9 的 Language Update 文档中:

```
More Concise try-with-resources Statements 更简洁的 try-with-resources 语句

Small Language Changes in Java SE 9 Java SE 9 中的小型语言变更

	@SafeVarargs annotation is allowed on private instance methods @SafeVarargs 注解允许用于私有实例方法   
	
	You can use diamond syntax in conjunction with anonymous inner classes 菱形语法可与匿名内部类结合使用
	
	The underscore character is not a legal name 下划线字符不是合法名称
	
	Private interface methods are supported 支持私有接口方法
```

### 2.3 Java Core Library

Java 核心类库的最新信息, 例如 JDK21 的 Core Library 包含以下主题:

```
Serialization Filtering  序列化过滤

Enhanced Deprecation  增强型弃用机制

XML Catalog API XML 目录 API

Java Collections Framework Java 集合框架

Process API  进程 API

Preferences API  首选项 API

Java Logging Overview  Java 日志记录概述

Java NIO

Java Networking  Java 网络编程

Pseudorandom Number Generators 伪随机数生成器

Foreign Function and Memory API 外部函数与内存 API

Scoped Values  作用域值

Concurrency  并发编程
```

## 3 JEP (JDK Enhancement Proposal)

```embed
title: "JEP 0: JEP Index"
image: "https://openjdk.org/images/openjdk2.svg"
description: ""
url: "https://openjdk.org/jeps/0"
favicon: ""
aspectRatio: "27.12871287128713"
```


即 JDK 增强提案, 是一份正式的文档, 用于提议, 讨论和定义 JDK 的新增功能和重大改进.

是一些非正式功能的 "毕业证书", 许多特性开始会以实验性或者预览功能出现, 等到特性足够成熟, 则会通过一个 JEP 将其最终定稿, 成为 Java 的正式标准功能.

可以从中获得关于该特性的提出动机, 目标, 描述, 依赖关系等信息; 通过 JEP 可以深入了解整个新特性的来龙去脉, 设计理念和最佳实践.

## 4 JSR (Java Specification Request)

```embed
title: "overview"
image: "https://www.jcp.org/favicon.ico"
description: "JSRs: Java Specification Requests"
url: "https://www.jcp.org/ja/jsr/overview"
favicon: ""
aspectRatio: "100"
```

即 Java 规范请求, 是正式的流程文档, 旨在提议和发展一项新的 Java 技术标准或者对现有标准进行修订;

一个 JSR 可以包含多个 JEP 来实现, 比如 [JSR 376: Java Platform Module System](https://www.jcp.org/en/jsr/detail?id=376) 模块化系统规范请求, 是通过多个 JEP 来实现的, 例如 [JEP 261: 模块系统(核心实现)](https://openjdk.org/jeps/261), [JEP 200: 模块化JDK](https://openjdk.org/jeps/200) 等等;

包含了当前标准的目标, 阶段, 时间表, 参与团队, 技术描述等信息.

## 5 OpenJDK Releases

```embed
title: "JDK"
image: "https://openjdk.org/images/openjdk2.svg"
description: ""
url: "https://openjdk.org/projects/jdk/"
favicon: ""
aspectRatio: "27.12871287128713"
```

### 5.1 背景补充

某个一个版本的 JDK 的发布流程大概是这样的:

1. 新功能的开发在 [OpenJDK](https://github.com/openjdk) 项目的各个子项目 (如 jdk, jmc 等) 中公开进行的, Oracle 是最大的主导者和贡献者;
2. 当开发周期接近尾声的时候, OpenJDK 会创建一个代码冻结分支, 如 `jdk21`, 这个分支只接收严重 bug 修复, 而不会再添加新功能;
3. 此时, 各个厂商 (如 Oracle, Amazon, Azul 等), 会获取源代码, 进行构建测试, 添加差异化内容等操作来构建发行版;
4. 之后, 各个厂商会进行 JDK 的支持和维护, 提供基于 OpenJDK 源码的 LTS 发行版, 并提供一些付费支持服务.

和 Linux 内核与 Linux 的关系类似, OpenJDK 类似 Linux 的内核, Oracle JDK, Amazon Corretto 等就类似于 Ubuntu, RHEL 这样的发行版本.

Oracle 于 2010 年收购了 Sun Microsystems 公司, 获取到了 Java 的核心技术, 商标和知识产权等;

相较于其他公司, Oracle 拥有这些特殊点:

1. 主导 Java 技术演进和标准规范, 掌控发展方向.
2. 拥有 Java 商标, 只有其官方实现能冠以“Java”之名, 其他厂商只能使用 OpenJDK 或自己的品牌 (如 Corretto, Zulu).
3. 为其 Oracle JDK 提供需要付费订阅的长期支持 (LTS), 将免费支持作为吸引用户升级的策略, 从而驱动其商业业务.
4. 是 OpenJDK 项目的主要维护者和贡献者, 对核心代码拥有最大的影响力.

### 5.2 内容概览

OpenJDK 的发布文档是以 JEP 主导的, 可以快速直观的看到当前版本更新引入了哪些 JEP; 如果想要了解更多信息需要去阅读具体的 Release 文档或者 Oracle Documention.

## 6 Inside Java

```embed
title: "Inside.java"
image: "https://inside.java/images/java-cup.png"
description: "News and views from members of the Java team at Oracle"
url: "https://inside.java/"
favicon: ""
aspectRatio: "93.75"
```


Inside Java 提供来自 Oracle Java 团队成员的新闻和观点, 它直接提供来自 Java Platform Group 的更新; 包括新闻广播、播客、简报和文章, 涵盖 Java 语言开发、JVM 更新、平台安全、创新项目以及社区活动等主题. 

作为一个官方渠道, 用于分享洞察、技术深入探讨以及与 Java 技术和生态系统相关的公告. 这个资源帮助开发人员直接从 Java 生态系统的创建者那里了解 Java 生态系统的最新进展和持续工作.