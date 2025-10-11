---
type: Rust
sub-type: Ownership
---
## 1 生命周期

一个变量的生命周期就是从它创建到销毁的整个过程:

```rust
fn main() {
    let v: i32 = 10; // lifetime of v starts here
    {
        let x = v + 1; // lifetime of x starts here
        println!("{}", x); // lifetime of x is ended here
    }
    println!("{}", v); // lifetime of v is ended here
}
```

## 2 介绍

变量对其管理的内存拥有所有权, 这个所有权不仅可以被 move, 还可以被 borrow;

借用指针使用 `&` 或 `&mut` 表示, 前者表示只读借用, 后者表示读写借用; 借用指针与普通指针的内部数据是一模一样的, 但是在语义层面, 借用指针对指向的内存**没有所有权**;

```rust
fn foo(v: &mut Vec<i32>) {
    v.push(10);
}

fn main() {
    let mut v = vec![1, 2, 3];
    foo(&mut v);
}
```

Rust 中的指针是一个数据类型, 所以使用 `mut` 修饰的指针变量是可以被重新绑定的; 借用指针在编译后, 实际上就是一个普通指针, 它的意义只是在编译阶段的静态检查中.

## 3 借用规则

对于借用指针, 有以下几个规则:

- 借用指针不能比它指向的变量存在的时间更长;
- `&mut` 型借用只能指向本身具有 mut 修饰的变量, 对于只读变量, 不能有可写的借用;
- `&` 指针能够同时存在多个, 而 `&mut` 型指针和其他借用型指针是互斥的, 会出现编译错误;