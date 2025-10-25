
Crates.io 是 Rust 的**官方包分发平台**, 主要分发开放源码, 将代码发布到 crates.io 可以与他人分享自己的代码, 让别人更方便地依赖你的库。

## 1	Making Useful Documentation Comments

用三斜杠 `///` 给公共 API 编写文档, 支持 Markdown 格式, 示例文档代码块会被 `cargo test` 作为测试运行, 确保示例是有效的.

```rust
/// 将一加到所给数字。
/// # Examples
///
/// ```
/// let arg = 5;
/// let answer = cargo_features_demo::add_one(arg);
///
/// assert_eq! (6, answer);
/// ```
pub fn add_one(x: i32) -> i32 {
    x + 1
}
```

Documentation Comments 常见的内容有:

- **Examples** 用例;
- **Panics** 何时会 panic;
- **Errors** 返回错误情况 (如果有 `Result`);
- **Safety** 如果是 `unsafe`, 需解释不安全原因及须遵守的条件.

## 2	代码箱、模组整体注释

用 `//!` 写包的整体文档, 描述 crate 或模块目标, 通常放在 `src/lib.rs` 顶部:

```rust
//! # Cargo 特性示例代码箱
//!
//! `cargo_features_demo` 是令到执行某些确切计算更便利
//! 的一些工具的集合。
//!
```

这些文档会展示在文档首页, 帮助用户理解整体结构.

