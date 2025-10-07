---
title: 从 Java 8 转换到 Java 11 - Azure
source: https://learn.microsoft.com/zh-cn/java/openjdk/transition-from-java-8-to-java-11
created: 2025-08-29
description: 本文档提供了将 Java 应用程序从 Java 8 迁移到 Java 11 的实用指南，涵盖了使用工具识别和解决潜在问题，以及在 Azure 环境中优化性能和兼容性的建议。
finished: "false"
tag: clipper
updated: 2025-09-27 22:34:06
---
两个用于探查潜在问题的工具:

- jdeprscan: 可查看是否使用了已弃用或已删除的 API
- jdeps: 