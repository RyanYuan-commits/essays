---
type: Rust
sub-type: Ownership
---
## 1	Core Feature

`Box<T>` 对它所指的堆上数据拥有 **独占所有权**, 这意味着一个数据智能有一个 Box 指向它; 当 `Box<T>` 本身离开其作用域时, 会自动调用 `drop` 方法释放指向堆的内存, 这保证了绝对不会发生内存泄漏; `Box<T>` 实现了 `Deref` 和 `DerefMut` Trait, 所以你可以像使用普通引用一样, 使用 `*` 操作符来访问它指向的数据;

`Box<T>` 通常读作 "Box of T", 是 Rust 中最简单的智能指针, 它的核心功能只有一个: **将数据存储在堆上**, 其核心使用场景包括:

- 在需要精确大小的上下文中, 使用在编译时无法明确大小的类型;
- 在不使用拷贝的前提下, 转移大数据的所有权;
- 拥有某个值, 但是不在意它的具体类型, 仅在意其是否实现了某个 Trait 的情况.

## 2	Enabling Recursive Types with Boxes

```rust
// 编译错误！
enum List {
    Cons(i32, List), // Cons 节点包含一个 i32 和另一个 List
    Nil,             // Nil 节点代表列表结束
}

// fix it
enum List {
    Cons(i32, Box<List>), // Cons 节点包含一个 i32 和一个指向 List 的指针
    Nil,
}

fn main() {
    // 创建一个列表： (1 -> (2 -> Nil))
    let list = List::Cons(1, Box::new(List::Cons(2, Box::new(List::Nil))));
}
```

上面的代码, 编译器会报错, 因为它无法计算 List 的大小, `size_of(List) = size_of(32) + size_of(List)`, 而 Box 指针的大小是确定的, 可以使用 Box 来包装 List.

## 3	Transfering Ownership of a Large Amount of Data

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

如果一个结构体非常大, 在栈上创建和移动它会导致大量的内存拷贝, 影响性能, 此时, 我们就可以考虑将其放在堆上;

## 4	Using Trait Objects that Allow for Values Different Types

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

如果希望创建一个集合, 里面存放着不同类型, 但是实现了同一个 Trait 时, 需要使用 `Box`; 假设你有一个 `Draw` Trait,  `Circle` 和 `Square` 都实现了它; 你不能直接写 `Vec<Draw>`, 因为 `Draw` 是一个 Trait, 不是一个具体类型, 它没有确定的大小;
