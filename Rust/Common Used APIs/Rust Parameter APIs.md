---
type: Rust
sub-type: APIs
---
## 1	标准库 std::env::args

### 1.1	核心 API

```rust
// 功能: 返回一个命令行参数的迭代器, 元素类型为 String
std::env::args -> Args
```

第 0 个参数为当前程序的路径或名称; 后续参数是用户输入的各个参数, 使用空格进行分割.

在任何平台上都能工作, 并且确保参数被解析成有效的 Unicode String, 如果遇到无效的 Unicode, 在非 Windows 平台上会产生 Panic, 在 Windows 平台上会使用替代字符.

---

```rust
// 返回一个命令行参数的迭代器, 类型为 OsString 
std::env::arg_os() -> ArgsOs
```

当处理可能包含非 Unicode 字符的参数时, 例如在 Windows 上或某些文件路径中, 在跨平台 CLI 工具中非常重要, 好处是不会因为无效 Unicode 而产生 Panic, 但是处理起来更加麻烦.

## 2	第三方库: CLAP

CLAP (Command Live Argument Parser) 是 Rust 生态中事实上的标注, 功能强大, 性能优异, API 友好, 它提供了两种主要的定义参数的方法, 派生宏 和 Builder 模式, 可以通过下面的方式引入:

```rust
// cargo.toml
[dependencies]
// 使用 derive 特性以使用派生宏 
clap = { version = "x.x", feature = ["derive"] }
```

### 2.1	Derive API

#### 2.1.1	核心思想与简单示例

CLAP 的 Derive API 的核心思想的: 你定义一个 Struct, 它的字段就代表你的命令行参数.

比如, 当我们创建一个简单的程序, 这个程序接收一个必须的 name 参数, 和一个可选的 --count 标志:

```rust
use clap::Parser;

/// 一个简单的 CLI 程序, 用来打招呼
#[derive(Parser, Debug)]
#[command(version, about, long_about = None)]
struct Args {
    /// 要打招呼的人的名字
    #[arg(short, long)]
    name: String,

    /// 打招呼的次数
    #[arg(short, long, default_value_t = 1)]
    count: u8,
}

fn main() {
    let args = Args::parse();

    for _ in 0..args.count {
        println!("Hello, {}!", args.name);
    }
}
```

#### 2.1.2	常用的 arg 参数

| 属性                     | 说明                   |
| ---------------------- | -------------------- |
| `short = 'c'`          | 指定一个短标志              |
| `long = "config"`      | 指定一个长标志, 默认是字段名      |
| `default_value = "1"`  | 设置默认值. 字符串类型         |
| `default_value_t = 1 ` | 设置默认值, 类型安全          |
| `action = ...`         | 定义标志的行为              |
| `env = "MY_APP_PORT"`  | 允许从环境变量中读取内容         |
| `value_name= "FILE"`   | 在帮助信息中为值指定一个名称       |
| `(Doc Comment)`        | Rust 的文档注释会被自动用作帮助信息 |

#### 2.1.3	特殊参数类型

如果一个字段是 bool 类型, 它会自动成为一个标志, 出现表示 true, 否则为 false

```rust
#[arg(short, long, default_calue_t = false)]
debug: bool,

// cargo run --debug, 则 debug 参数会是 true
```

