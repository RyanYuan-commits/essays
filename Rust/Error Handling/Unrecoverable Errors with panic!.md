---
type: Rust
sub-type: ErrorHandling
---
程序中的 Error 分为 Recoverable Errors 和 Unrecoverable Errors, `panic!` 用于处理 Unrecoverable Errors. 当程序检测到一个致命的、违反内部逻辑一致性的 bug 时, 与其冒险继续运行在未知的危险状态下, 不如立即、干净地终止当前操作, 防止造成更大的破坏.

## 1	panic! Macro

`panic!` 宏是你用来手动触发 panic 的语言工具, 它接受一个字符串参数, 用来描述 panic 的原因:

```rust
panic!("divisor cannot be zero!")
```

当触发 panic 时, Rust runtime 会采取两种策略之一:

- Unwinding 栈展开策略: 当 panic 发生时, Rust 会想多米诺骨牌一样, 从当前函数开始, 沿着函数的调用栈弹出, 在每一层它都会负责任的运行该函数所有变量的析构代码, 释放内存, 关闭文件等, 这确保了资源被正确清理.
- Abort 终止策略: 这是更轻量的策略, 当 panic 发生时, 程序不进行任何清理, 而是直接向操作系统发出信号, 立即终止整个进行, "崩溃" 速度更快, 资源由操作系统来清理.

默认是 Unwinding 策略, 如果需要配置 Abort, 要在 `Cargo.toml` 中添加: 

```toml
[profile.release]
panic = 'abort'
```

## 2	Core Principle

你的代码直接调用 `panic!("...")`, 或者执行了一个会隐式触发 panic 的操作;

Rust Runtime System 会接管控制权;

执行具体的 Unwinding 或者 Abort 策略;

在清理或者终止之前, 默认的 panic 钩子函数会被调用, 它会将 panic 有关的各种信息打印到标准错误输出.

