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

## 1	The Why

Rust 使用 Borrow Checker 和 Lifetimes 来解决 [[Basic of Memory Management#3.1 Dangling Reference|Dangling Reference]] 的问题, 确保一个 Reference 的存活时间, 永远不能超过它指向数据的存活时间; Lifetimes 是一个编译器的分析工具, Rust 在编译时, 会分析和每个引用的有效范围, 确保其内存安全性.

## 2	The What

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

## 3	生命周期标记的必要性

**消除悬垂引用**:  在 C/C++ 自动内存管理中, 很容易返回一个指向已经被释放的局部变量的指针, 导致未定义的行为; 而在 Rust 中, 生命周期标记在编译时强制执行规则, 如果编译器无法证明返回的引用在原始数据被销毁后不会再被使用, 代码就会报错.

**告诉编译器数据依赖关系**: 编译器可以在很多简单的场景下**推断**生命周期, 但当函数签名变得复杂, 包含多个输入引用时, 编译器需要您的帮助来确定: **返回的引用**应该"借用"自哪个输入参数? 结构体中的引用字段, 存活时间应该**多久**?

## 4	Lifetime Elision Rules

在以下三种模式下, Rust 编译器会遵守生命周期省略规则, 此时无需用户手动输入声明周期参数:

- 如果一个函数签名中包含多个引用作为输入, 每个输入都会被自动分配一个不同的生命周期占位符;
- 如果只有一个引用输入, 那么这个引用的生命周期会赋给所有输出引用参数;
- 成员方法方法(即有 `&self` 或 `&mut self` 作为输入) 的 `self` 的生命周期被赋给所有输出引用参数.

## 5	Lifetimes in Struct and Impl

### 5.1	Struct Definition

为了将结构体的存活时间与其内部引用指向数据的存活时间关联起来, 如果一个结构体包含任何 Reference, 那么它必须定义时包含生命周期参数;

```rust
// 错误示例：缺少生命周期参数。编译器不知道它所引用的数据能存活多久。
// struct BadDataHolder {
//     data: &str
// }

// 正确示例：使用生命周期参数 'a
struct DataHolder<'a> {
    // 字段 'data' 必须至少存活 'a 这么久
    data: &'a str
}

fn main() {
    let s: String = String::from("我是一个字符串");
    
    { // 内部作用域开始
        // 结构体实例 'd' 的生命周期不能超过 's'
        let d = DataHolder { data: s.as_str() }; 
        // d 在此作用域内有效
    } // d 和 s.as_str() 的借用在这里结束

    // 假设 s 在这里被销毁 (Drop)
}
```

编译器会检查结构体实例的存活时间是否在生命周期的限制中, 如果结构体尝试突破限制, 编译器会报错.

### 5.2	Impl Stack

为了告诉编译器 impl 块中的生命周期和结构体的生命周期的关联, 在为包含了生命周期的结构体编写 `impl` 块的时候, 你必须在 `impl` 后面重复制定结构体定义时使用的生命周期参数.

```rust
// 结构体定义时使用了生命周期 'a
struct DataHolder<'a> {
    data: &'a str
}

// 在实现块 impl 后面，也必须指定 'a
impl<'a> DataHolder<'a> {
    // 1. 方法的签名（不需要额外生命周期）：
    //    由于生命周期省略规则（Rule 3），&self 的生命周期（'a）会自动应用于返回的引用。
    fn get_data(&self) -> &str {
        self.data
    }
    
    // 2. 构造函数
    fn new(s: &'a str) -> DataHolder<'a> {
        DataHolder { data: s }
    }
}
```

## 6	应用场景

**函数签名**, 如上面的 `longest`, 用于返回一个输入参数的引用;

**结构体定义**, 当结构体包含引用时, 必须包含生命周期, 确保结构体本身不会比它引用的数据存活更久:

```rust
// struct DataHolder { data: &str } // 错误：缺少生命周期
struct DataHolder<'a> { 
    data: &'a str 
}
```

