---
type: Rust
sub-type: Ownership
---
## 1	移动语义

一个变量可以把它拥有的值转移给另一个变量, 称为所有权转移, 赋值语句, 函数调用, 函数返回都有可能导致所有权转移:

```rust
fn create() -> String {
    let s = String::from("Hello, world!");
    s // 所有权转移
}

fn consume(s: String) {
    println!("{}", s);
}

fn main() {
    let s = create(); // s 获得所有权
    consume(s); // 所有权转移到 consume 函数
    // println!("{}", s); // 这里会报错，因为 s 的所有权已经被转移
}
```

Rust 中所有权转移的重要特点是: 它是所有类型的**默认语义**.

## 2	复制语义

### 2.1	默认复制语义

默认的 move 语义是 Rust 一个很重要的设计, 但是任何时候复制都需要手动调用 clone 显得非常繁琐; 所以, 对于一些简单的类型(如整数, bool), 在赋值时采用的是复制语义:

```rust
fn main() {
    let v1: isize = 0;
    let v2 = v1;
    println!("{}", v2);
    println!("{}", v1); // 可以正常执行
}
```

### 2.2	Clone & Copy

Rust 中, 在普通变量绑定, 函数传参, 模式匹配等场景下, 凡是实现了 `std::marker::Copy` 的类型, 都会执行 copy 语义, 基本类型, 比如数字, 字符, bool 等, 都实现了 Copy trait, 因此具备 copy 语义.

对于那些无法实现 `Copy` trait 的类型, 例如包含堆分配数据的 `String` 或 `Vec`, Rust 默认执行 move 语义. 如果我们需要这类类型的一个独立副本, 而非所有权转移, 我们就需要使用 `std::marker::Clone` trait. `Clone` trait 提供了一个 `clone()` 方法, 允许我们显式地创建一个值的深度复制品, 从而确保原始值和新副本各自拥有独立的数据.

```rust
struct Foo {
    data: i32,
}

impl Clone for Foo {
    fn clone(&self) -> Foo {
        Foo { data: self.data }
    }
}

impl Copy for Foo {}

fn main() {
    let f1 = Foo { data: 42 };
    let f2 = f1;
    println!("{:?}", f1.data); // This works because Foo implements Copy
    println!("{:?}", f2.data);
}

// === Simplify to this ===

#[derive(Copy, Clone, Debug)]
struct Foo;

fn main() {
    let f1 = Foo {};
    let f2 = f1;
    println!("{:?}", f1); // This works because Foo implements Copy
    println!("{:?}", f2);
}
```

`Copy` 和 `Clone` 的主要区别在于它们的语义和执行方式. `Copy` 是一个隐式的, 按位复制的操作, 仅适用于那些完全存储在栈上的数据. 而 `Clone` 则是一个显式的操作, 可以处理更复杂的数据结构, 包括堆数据, 并执行深度复制. 

值得注意的是, 任何实现了 `Copy` trait 的类型也必须实现 `Clone` trait, 因为 **Copy 只是 Clone 的一种特殊且高效的实现形式**. 通常, 当一个类型可以实现 `Copy` 时, 应当优先选择 `Copy`, 因为它通常开销更小.

