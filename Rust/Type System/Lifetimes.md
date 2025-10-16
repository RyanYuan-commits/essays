---
type: Rust
---

```embed
title: "Validating References with Lifetimes - The Rust Programming Language"
image: "https://doc.rust-lang.org/book/img/ferris/does_not_compile.svg"
description: ""
url: "https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html"
favicon: ""
aspectRatio: "67.27561556791105"
```



## 1	The What

Rust Lifetimes 用于告诉 Rust 的 Borrow Checker 引用和它指向的数据之间的存活时间关系, 当一个函数或者结构体包含多个引用的时候, 生命周期标记用于 **将这些引用的存活时间关联起来**.

```rust
// 错误示例: 编译器不知道返回的引用应该依赖于哪个输入引用
// fn longest(x: &str, y: &str) -> &str { ... } 

// 正确示例：使用生命周期标记关联输入和输出
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    // 借用检查器知道: 返回的引用的存活时间不能超过 x 或 y
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

在上面的案例中, `x: &'a str` 表示输入 `x` 至少存活 `'a` 这么久, 而 `-> &'a str` 表示返回的引用也要存活 `'a` 这么久; 最终, 编译器可以保证调用上面函数获取的引用, 在 x 或 y 被销毁后, 无法继续使用.

## 2	生命周期标记的必要性

**消除悬垂引用**:  在 C/C++ 自动内存管理中, 很容易返回一个指向已经被释放的局部变量的指针, 导致未定义的行为; 而在 Rust 中, 生命周期标记在编译时强制执行规则, 如果编译器无法证明返回的引用在原始数据被销毁后不会再被使用, 代码就会报错.

**告诉编译器数据依赖关系**: 编译器可以在很多简单的场景下**推断**生命周期, 但当函数签名变得复杂, 包含多个输入引用时, 编译器需要您的帮助来确定: **返回的引用**应该"借用"自哪个输入参数? 结构体中的引用字段, 存活时间应该**多久**?

## 3	Lifetime Elision Rules

在以下三种模式下, Rust 编译器会遵守生命周期省略规则, 此时无需用户手动输入声明周期参数:

- 如果一个函数签名中包含多个引用作为输入, 每个输入都会被自动分配一个不同的生命周期占位符;
- 如果只有一个引用输入, 那么这个引用的生命周期会赋给所有输出引用参数;
- 如果函数是一个方法(即有 `&self` 或 `&mut self` 作为输入), 则 `self` 的生命周期被赋给所有输出引用参数.

## 4	应用场景

**函数签名**, 如上面的 `longest`, 用于返回一个输入参数的引用;

**结构体定义**, 当结构体包含引用时, 必须包含生命周期, 确保结构体本身不会比它引用的数据存活更久:

```rust
// struct DataHolder { data: &str } // 错误：缺少生命周期
struct DataHolder<'a> { 
    data: &'a str 
}
```