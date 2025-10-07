---
title: 引言 - Rust 语言中的 Hypervisor 入门
source: https://tandasat.github.io/Hypervisor-101-in-Rust/introduction/index.html
created: 2025-09-18
description: 这是一门为期一天的 Rust 语言实战课程，教授如何利用硬件虚拟化技术构建用于高性能模糊测试的 Hypervisor。
finished: "false"
tag: clipper
updated: 2025-09-27 22:34:06
---
需要补充学习的知识：Rust 语言基础、硬件辅助虚拟化原理（Intel VT-x/AMD-V）、虚拟机管理程序（Hypervisor）架构、模糊测试（Fuzzing）技术。

关键词：Hypervisor、Rust、硬件虚拟化、VMCS、EPT、模糊测试

详细总结：这是一门为期一天的实践课程，专注于使用 Rust 语言编写高性能虚拟机管理程序（Hypervisor）的核心技术，重点涵盖硬件辅助虚拟化基础（如 VMCS/VMCB、EPT/NPT、世界切换）以及利用异常拦截实现虚拟机自省，以支持模糊测试场景。课程包含理论讲解和动手编程练习，需使用特定 Git 分支（gcc2023）的代码库进行逐步实践。