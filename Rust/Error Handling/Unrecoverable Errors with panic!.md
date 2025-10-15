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

当 panic 发生的时候