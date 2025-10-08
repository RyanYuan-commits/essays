---
type: Rust
sub-type: Basic
---
Rust 官方网站给自己的介绍是: Rust is a systems programming language that runs blazingly fast, **prevents segfaults**, and guarantees thread safety.

segfault 实际上是 segmentation fault 的缩写, 可以翻译为段错误, segfault 是这样形成的: 

- 进程空间中每个段通过硬件 MMU 映射到真正的物理空间;
- 在这个映射中, 我们还可以给不同的段设置不同的访问权限, 比如代码段就是只能读不能写;
- 进程在执行这个的过程中, 如果违反了这些权限, CPU 会直接产生一个硬件异常;
- 硬件异常被操作系统内核处理, 内核会向进程发送一条信号;
	- 如果应用没有实现信号处理函数, 默认情况下, 这个进程会直接正常退出
	- 如果操作系统打开了 core dump 功能, 在进程退出的时候操作系统会把当时的内存状态, 寄存器状态, 以及各种相关信息保存下来;

在传统系统级编程语言 C/C++ 中, 制造 segfault 是很容易的, 程序员要非常小心才能避免这种错误;另一类规避 segfault 的语言, 使用的是自动内存回收机制, 在这些编程语言中, 指针的能力被大幅度限制, 内存被放在一个运行时环境中严格管理;

Rush 主要设计目标之一就是在不用自动垃圾回收机制的前提下, 避免产生 segfault.



