---
type: Rust
sub-type: ProjectManagement
---
Cargo 是 Rust **官方构建系统和包管理器**, 用于管理项目依赖, 编译代码, 运行测试和生成文档.

## 1	Core Components

**Cargo.toml 项目清单文件**: 这是 Cargo 的大脑和智慧中心, 是一个 TOML 格式的文本文件, 定义了项目的元信息, 其中最核心的是:

- `[package]`: 项目名, 版本, 作者的基本信息;
- `[dependencies]`: 项目需要以来的外部库 (在 Rust 中称为 crate) 及其版本要求.

---

**Cargo.lock 版本锁定文件**: Cargo 自动生成的文件, 不要手动修改!!!  当第一次构建项目的时候, Cargo 会计算出所有依赖的一个精确版本组合, 并将这个 snapShoot 记录在 `Cargo.lock` 文件中, 确保了可复现构建(Reproducible Build).

---

**[crates.io](http://crates.io) 中央仓库**: Rust 社区的官方包注册中心, 就像 Python 的 Pypl, Node.js 的 npm, Cargo 会默认从这里查找并下载你需要的依赖.

## 2	Commands

```sh
# 创建一个新的 Rust 项目模板
cargo new 

# 编译项目
cargo build

# 编译并运行项目
cargo run

# 运行项目中的所有测试
cargo test

# 快速检查代码中是否有编译错误, 但生成可执行文件, 速度比 build 快
cargo check

# 将自己的库发布到 crates.io
cargo publish
```