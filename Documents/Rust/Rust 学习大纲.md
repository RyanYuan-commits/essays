---
created: 2025-10-05 15:53:29
updated: 2025-10-05 15:53:52
---
## 01-基础语法与工程实践

### 基础语法

- 变量声明
- 基本数据类型
- 复合数据类型 (tuple, struct, enum)
- 语句与表达式 (if-else, loop, for)
- 函数 (fn, const fn)
- trait (定义、方法、约束、继承)
- 数组与字符串
- 模式解构 (match, if-let, while-let)

### 工程化工具链

- Cargo 包管理
- 项目与模块组织 (crate, mod)
- 文档与测试

## 02-所有权与内存安全

### 内存管理基础

- 堆和栈
- 段错误 Segfaults
- 内存安全

### 核心内存机制

- 所有权 Ownership
- 移动语义 Move Semantics
- 复制语义 Copy Semantics
- 借用 Borrowing
- 生命周期 Lifetime 标记
- Box 智能指针

### 借用检查与安全边界

- 借用检查规则 (共享不可变, 可变不共享)
- NLL 非词法生命周期
- 解引用 Deref 与自动转换
- 析构函数 Drop trait

## 03-高级抽象与类型系统

### 类型系统与错误处理

- 代数类型系统 ADT
- Option Result 类型
- Never Type (发散类型 !)
- 宏 Macro (示范型、过程宏)

### 泛型与行为抽象

- 泛型 Generics
- Trait 定义与约束
- 关联类型
- 泛型特化 Specialization

### 函数与动态抽象

- 闭包 Closure (变量捕获, move)
- Fn FnMut FnOnce Traits
- 动态分派 Trait Object
- 静态分派 impl Trait

## 04-并发与状态共享

### 线程安全基础

- 线程 Thread 启动与 Join
- 免数据竞争 Data Race
- Send Sync Marker Trait
- 线程局部存储 Thread Local

### 多线程状态共享

- Arc (Atomic Reference Counting)
- Mutex 互斥锁
- RwLock 读写锁
- Atomic 原子类型
- 死锁 Deadlock
- 条件变量 Condvar

### 线程通信与扩展

- 异步管道 mpsc (Multi-producer single-consumer)
- 同步管道
- 第三方并行库 (Rayon, crossbeam)

## 05-高级工具与底层编程

### 内部可变性与底层控制

- 内部可变性 Interior Mutability
- Cell (Copy, Set, Get)
- RefCell (Borrow, BorrowMut)
- UnsafeCell
- Unsafe 关键字与代码块
- 裸指针 Raw Pointer

### 错误处理与抽象深化

- 错误类型 Result Option
- 问号运算符 ?
- 错误组合与 Failure 库

### 外部函数接口 FFI

- FFI 概念与 C ABI 兼容
- extern C 与 no mangle
- 复杂数据类型与 repr C

### 容器与迭代器

- Vec VecDeque HashMap BTreeMap
- 迭代器 Iterator 与惰性求值
- 生成器 Generator 与协程 Coroutine