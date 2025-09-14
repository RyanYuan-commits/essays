---
type: Rust
sub-type: Basic
---

Rust 变量管理的核心是安全和并发, 变量默认是不可变的(immutable).

## 1 默认不可变性 Immutability

Rust 中, 变量使用 `let` 关键字声明:

```rust
fn main() {
    let x = 5;
    println!("x 的值为: {}", x);
    
    // 下面这行代码会报错，因为 x 是不可变的
    x = 6; 
}
```

使用 `let` 声明的变量是不可变的, 这意味着, 一旦给一个变量赋值, 就不能再改变它的值.

在多线程环境下, 声明为 `let` 的变量可以被安全读取, 无需担心数据竞争.

## 2 可变性 Mutability

如果需要修改一个变量的值, 需要使用 `mut` 关键字来使其变为可变的:

```rust
fn main() {
    // 使用 mut 关键字声明可变变量
    let mut y = 5;
    println!("y 的初始值为: {}", y);

    // 现在可以改变 y 的值了
    y = 6;
    println!("y 改变后的值为: {}", y);
}
```

## 3 变量遮蔽 Shadowing

Rust 允许你使用相同的变量名来声明一个新年两, 这个新变量会遮蔽掉之前的同名变量.

```rust
fn main() {
    let z = 5;
    println!("z 的初始值为: {}", z);

    // 遮蔽之前的 z
    let z = z + 1;
    println!("z 遮蔽后的值为: {}", z);

    {
        // 内部作用域的遮蔽
        let z = z * 2;
        println!("内部作用域中 z 的值为: {}", z);
    } // 内部作用域结束，z 恢复为外部的值

    println!("外部作用域中 z 的值为: {}", z);
}
```

与 Mutability 不同的是, Shadowing 可以**修改变量的类型**.

## 4 常量 Constants

使用 constant 声明, 使用全大写字母命名, 单词之间使用 `_` 分割.

常量必须显示的声明类型, 且永远不可变.

```rust
// 在全局作用域声明常量
const THREE_HOURS_IN_SECONDS: u32 = 60 * 60 * 3;

fn main() {
    let x = 5;
    const GREETING: &str = "Hello, Rust!";

    println!("常量值为: {}", THREE_HOURS_IN_SECONDS);
    println!("问候语为: {}", GREETING);
}
```



