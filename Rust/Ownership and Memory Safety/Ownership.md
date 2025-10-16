---
type: Rust
sub-type: Ownership
created: 2025-10-09 23:33:11
updated: 2025-10-09 23:33:11
---
所有权是一套用来**管理 Rust 如何操作内存的规则**, 一些语言使用 grabage collection 来周期性的寻找和清理不再被使用的内存, 一些语言则需要程序员 explicit allocate and free the memory, 而 Rust manages memory through a system of ownership with a set of rules that the compiler checks; 如果违反了任何规则, 程序将无法编译.

比如, 在 C 语言中, 我们会在堆上创建一个对象, 然后用一个指针来管理这个对象, 接下来, 我们可能要使用这个对象: 

```cpp
Foo *p = make_object("args");
use_object(p);
```

然而, 在这段代码之后, 指针指向的这个对象到底发生了什么, 是不可预测的; 它是否被修改过? 他是否还存在? 我们还能继续使用这个对象吗? 除了去阅读 `use_object()` 这个方法以外, 我们没法回答以上的问题; 

对此, C++ 通过智能指针来描述所有权的概念, 实现了 "半自动化" 的内存管理; 而 Rust 更进一步, **将所有权理念直接融入到语言中**.

## 1	Ownership Rules

所有权代表代表以下含义:

- Each value in Rust has an owner;
- There can only be one owner at a time;
- When the owner goes out of scope, the value will be dropped.

```rust
fn main() {
	let mut s = String::from("hello");
	let s1 = s;
	println!("{}", s);
}
```

上面的代码会出现编译错误, 因为在 `let s1 = s;` 语句中, 原本由 s 拥有的字符串已经转移给了 `s1`, 后续就无法继续使用 s;

而在 C++, 使用 `s` 初始化 `s1` 调用 `string` 类型的复制构造函数复制一个新的字符串, 在 Rust 中如果要实现这种行为, 需要手动调用 `clone()` 函数:

```rust
fn main() {
    let mut s = String::from("hello");
    let s1 = s.clone();
    s.push_str(", world");
    println!("{}", s);
    println!("{}", s1);
}
```

在 Rust 中, 不可做 "赋值运算符重载", 如果要实现 "深复制", 必须手动调用来自 `std::clone::Clone` trait 的 `clone` 方法.

## 2	Variable Scope

```rust
{                      // s is not valid here, since it's not yet declared
    let s = "hello";   // s is valid from this point forward

    // do stuff with s
}                      // this scope is now over, and s is no longer valid
```

当 s 进入作用域时, 它开始生效, 直到离开作用域为止.

## 3	Memory and Allocation

对于字符串字面量, 在编译时其内容是确定的, 因此文本会直接硬编码到 executable 文件, 所以字符串字面量快速且高效;

但是 [[String]] 类型, 为了支持 mutable 和 growable 特性, 需要在堆上分配一块在编译时未知的大小的内存来存放这个 String, 这意味着:

- 必须在运行时向内存分配器请求内存;
- 使用完 `String` 后, 需要用一种方式将内存返回给分配器.

第一部分由我们调用 `String::from` 来完成; 第二部分由 Rust 来完成, 当拥有内存的变量离开作用域, 其内存会被自动回收.

```rust
{
    let s = String::from("hello"); // s is valid from this point forward

    // do stuff with s
}                                  // this scope is now over, and s is no longer valid
```

当变量离开作用域时, Rust 会调用一个 [[Drop trait]] 函数, String 的作者可以在这里编写归还内存的代码. 

## 4	Ownership and Functions

将值传递给函数的机制和将值赋值给变量的机制类似, 同样会 move or copy;

```rust
fn main() {
    let s = String::from("hello");  // s comes into scope
    
    takes_ownership(s);             // s's value moves into the function and so is no longer valid here

    let x = 5;                      // x comes into scope

    makes_copy(x);                  // Because i32 implements the Copy trait, x does NOT move into the function

}

fn takes_ownership(some_string: String) {
    println!("{some_string}");
}

fn makes_copy(some_integer: i32) {
    println!("{some_integer}");
}
```

## 5	Return Values and Scope

返回值的操作同样会转移所有权;

```rust
fn main() {
    let s1 = gives_ownership(); // gives_ownership moves its return value into s1

    let s2 = String::from("hello"); // s2 comes into scope

    let s3 = takes_and_gives_back(s2);
}

fn gives_ownership() -> String {
    let some_string = String::from("hello");
    some_string
}

fn takes_and_gives_back(a_string: String) -> String {
    a_string
}
```

变量的所有权转移每次遵循相同的模式, 将值赋给另一个变量会移动它, 当包含堆上数据的变量离开作用域时, 除非所有权已经转移给了另一个变量, 否则该值将被清除.

---

每个函数都要获取所有权在返回所有权实在有些繁琐, 如果我们想让函数使用某个值但不获取所有权呢? 除了函数体想要传回的数据, 任何传入的值想要使用都必须要传回, it's quite annoying.

Rust 提供了一种无需所有权转移即可使用值的功能, 称为引用 [[References and Borrowing|Reference]].