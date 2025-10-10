---
type: Rust
sub-type: Ownership
---
Box 类型是 Rust 中常用的指针类型, 它代表 "拥有所有权的指针":

```rust
struct T {
    value: i32,
}

fn main() {
    let p = Box::new(T { value: 42 });
    println!("{}", p.value);
}
```

Box 类型执行的永远是 move 语义, 不能是 copy 语义, 执行 copy 语义进行浅复制很显然会引发二次释放;

Rust 中还有一个保留关键字 `box`, 用于将变量 "装箱" 到堆上: 