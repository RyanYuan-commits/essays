---
type: Rust
sub-type: ProjectManagement
---
Cargo 是 Rust **官方构建系统和包管理器**, 用于管理项目依赖, 编译代码, 运行测试和生成文档.
## 1	Core Components

### 1.1	[Cargo.toml](https://doc.rust-lang.org/cargo/reference/manifest.html#the-manifest-format)

项目清单文件, 这是 Cargo 的大脑和智慧中心, 是一个 [TOML](https://toml.io/cn/v1.0.0) 格式的文本文件, 定义了项目的元信息, 其中最核心的是:

- `[package]`: 项目名, 版本, 作者的基本信息;
- `[dependencies]`: 项目需要以来的外部库 (在 Rust 中称为 crate) 及其版本要求.

### 1.2	Cargo.lock 

版本锁定文件, Cargo 自动生成的文件, 不要手动修改!!!  当第一次构建项目的时候, Cargo 会计算出所有依赖的一个精确版本组合, 并将这个 snapShoot 记录在 `Cargo.lock` 文件中, 确保了可复现构建(Reproducible Build).

### 1.3	[crates.io](http://crates.io)

中央仓库, Rust 社区的官方包注册中心, 就像 Python 的 Pypl, Node.js 的 npm, Cargo 会默认从这里查找并下载你需要的依赖.

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

## 3	How It Works

- 当执行 `cargo build` 时, Cargo 会先读取 `cargo.toml` 文件, 了解你的项目信息和直接依赖;
	
- 然后, 它会检查 `Cargo.lock` 文件是否存在, 如果存在, 其会按照文件记录的精确版本号去下载依赖; 如果不存在, 它会根据 toml 文件的配置去 `crates.io` 查询, 找到所有依赖最新的兼容版本, 构建一个完整的依赖关系图, 并将这个精确的图写入新的 `Cargo.lock` 文件;
	
- 下载完所有依赖后, Cargo 会先编译依赖库, 然后编译仓库 src 目录下的代码;
	
- 所有的编译产物最终会统一存放在 target 目录中.
