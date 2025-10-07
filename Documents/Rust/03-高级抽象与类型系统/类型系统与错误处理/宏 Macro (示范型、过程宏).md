---
type: Rust
sub-type: Basic
created: 2025-10-05 16:00:19
updated: 2025-10-05 19:59:13
---

## 1 是什么?

宏是 Rust 元编程工具, 简单来说就是**编写代码的代码**;

函数在运行时处理值, 而宏在编译时处理代码本身.

## 2 宏的核心要素

### 2.1 规则

一个宏定义可以包含多个规则, 每个规则使用 `[匹配器] => [替换器]` 的形式定义:

```rust
macro_rules! my_macro {
    // 规则 1：匹配零参数调用
    () => { println!("No arguments!"); }; 

    // 规则 2：匹配一个表达式
    ($e:expr) => { println!("Argument: {}", $e); }; 
}
```

### 2.2 片段指示符

片段指示符 (Fragment Specifiers) 是宏匹配的 "数据类型", 你不能直接匹配 `String` 或 `i32`, 而是要匹配一个 **语法片段**.

| **指示符** | **匹配内容**                                    | **示例**                              |
| ------- | ------------------------------------------- | ----------------------------------- |
| `expr`  | 任何合法的 Rust 表达式(例如 `1 + 2`, `foo()`, `true`) | `map! { $key:expr => $value:expr }` |
| `ident` | 标识符(变量名, 函数名)                               | `fn $name:ident() {}`               |
| `ty`    | 类型(例如 `i32`, `String`, `Vec<T>`)            | `struct S<$t:ty>;`                  |
| `stmt`  | 单个语句(Statement)                             |                                     |
| `pat`   | 模式(Pattern，例如 `let Some($v:pat)`)           |                                     |

### 2.3 捕获变量

捕获变量(Captures), 在匹配器中, 使用 `$` 后面跟着名字来(如 `$key`) 来匹配捕获到的代码片段;

捕获变量可以在替换器中被引用来生成代码.

### 2.4 重复匹配

重复匹配(Repetitions), 用于处理列表或可变参数, 重复模式的核心结构为:

```rust
$( 模式 ) 分隔符? 重复符
```

分隔符是可选的, 默认用空格隔开, 常见的分隔符有 `,` 和 `;` 等;

重复符可选 `*` 或者 `+`, `*` 匹配 0 次或多次, `+` 匹配一次或多次.

## 3 Rust 的两种宏

### 3.1 声明式宏

最常见, 相对简单的一种宏, 通过 `macro_rules!` 来定义;

实现原理基于模式匹配, 检查输入的代码是否匹配某种模式, 如果匹配, 就将其转化为预先定义好的一段代码.

声明式宏的标志是以 `!` 结尾, 如 `pritln!`, `vec!`.

```rust
// 声明式宏案例
macro_rules! map {
    ($( $key:expr => $value:expr ),+) => {
        {
            let mut mut_map = std::collections::HashMap::new();
            $(
                mut_map.insert($key, $value);
            )*
            mut_map
        }
    };
    () => {
        println!("Empty map");
    }
}

fn main() {
    let my_map = map! {
        "one" => 1,
        "two" => 2,
        "three" => 3
    };
    println!("{:?}", my_map);
    map!();
}
```

### 3.2 过程宏

过程宏 (Procedural Macros) 是一种更强大的宏, 它在编译时接收并操作 Rust 代码的**词法单元流** (Token Stream), 允许进行更复杂的代码生成和转换.

与声明式宏不同, 过程宏更像一个函数, 输入是 `TokenStream`, 输出也是 `TokenStream`.

过程宏主要有三种类型:

**派生宏 (Derive Macros)**

- 用于为 `struct` 和 `enum` 自动实现 `trait`.
- 通过 `#[derive(TraitName)]` 属性来使用.
- 这是最常用的一种过程宏, 例如 `#[derive(Debug)]`, `#[derive(Clone)]`.

**属性宏 (Attribute-like Macros)**

- 可以附加到任何项 (item) 上, 例如函数, 模块等.
- 用于修改或检查附加到的项.
- 例如, 著名的 Web 框架 `actix-web` 中的 `#[get("/")]`.

**函数式宏 (Function-like Macros)**

- 看起来像函数调用, 但操作的是词法单元.
- 例如, `sqlx` 库中的 `sqlx::query!("SELECT * FROM users")` 宏, 它会在编译时检查 SQL 语法的正确性.