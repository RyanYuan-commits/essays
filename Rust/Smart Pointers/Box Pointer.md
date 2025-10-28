---
type: Rust
sub-type: Ownership
---
## 1 什么是 Box 指针?

`Box<T>` 通常读作 "Box of T", 是 Rust 中最简单的智能指针, 它的核心功能只有一个: **将数据存储在堆上**.

### 1.1 核心特性

`Box<T>` 对它所指的堆上数据拥有 **独占所有权**, 这意味着一个数据智能有一个 Box 指向它; 当 `Box<T>` 本身离开其作用域时, 会自动调用 `drop` 方法释放指向堆的内存, 这保证了绝对不会发生内存泄漏; `Box<T>` 实现了 `Deref` 和 `DerefMut` Trait, 所以你可以像使用普通引用一样, 使用 `*` 操作符来访问它指向的数据;

### 1.2 基本用法

```rust
fn main() {
    let b = Box::new(5);

    // Box implements the Debug trait
    println!("b = {:?}", b);

    // Copy the value out of the Box
    let value = *b; // Dereference the Box to get the value
    println!("value = {}", value);
}
```

## 2 使用场景

### 2.1 创建递归数据结构

Box 最经典的应用场景, 在 Rust 中, 所有类型在编译时必须有确定的, 已知的大小, 对于递归类型来说, 这是一个问题:

```rust
// 编译错误！
enum List {
    Cons(i32, List), // Cons 节点包含一个 i32 和另一个 List
    Nil,             // Nil 节点代表列表结束
}
```

上面的代码, 编译器会报错, 因为它无法计算 List 的大小, `size_of(List) = size_of(32) + size_of(List)`;

此时, 可以使用 Box 来解决:

```rust
enum List {
    Cons(i32, Box<List>), // Cons 节点包含一个 i32 和一个指向 List 的指针
    Nil,
}

fn main() {
    // 创建一个列表： (1 -> (2 -> Nil))
    let list = List::Cons(1, Box::new(List::Cons(2, Box::new(List::Nil))));
}
```

指针的大小是确认的, 此时 List 的大小就可以被确定.

### 2.2 使用 [[Trait]] Object

如果希望创建一个集合, 里面存放着不同类型, 但是实现了同一个 Trait 时, 需要使用 `Box`;

假设你有一个 `Draw` Trait,  `Circle` 和 `Square` 都实现了它; 你不能直接写 `Vec<Draw>`, 因为 `Draw` 是一个 Trait, 不是一个具体类型, 它没有确定的大小;

```rust
fn main() {
    let shapes: Vec<Box<dyn Draw>> = vec![
        Box::new(Circle { radius: 10.0 }),
        Box::new(Square { side: 5.0 }),
    ];

    for shape in shapes {
        shape.draw();
    }
}
```

### 2.3 大尺寸数据所有权转移

如果一个结构体非常大, 在栈上创建和移动它会导致大量的内存拷贝, 影响性能, 此时, 我们就可以考虑将其放在堆上:

```rust
struct HugeData {
    data: [u8; 1024 * 1024], // 1MB 的数据
}

fn process_data(data: Box<HugeData>) {
    // ...
}

fn main() {
    let huge_data = Box::new(HugeData { data: [0; 1024 * 1024] });
    // 这里只拷贝了一个指针（8字节），非常高效
    process_data(huge_data);
}
```