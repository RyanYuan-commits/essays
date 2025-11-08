---
type: Rust
sub-type: Smart Pointers
---
## 1	Interior Mutability

Rust 的借用检查规则的核心思想是 "共享不可变, 可变不共享", 蛋仔某些情况下需要在共享的情况下可变, 为了让这种情况是可控的, Rust 还设计了一种内部可变性 Interior Mutability.

Interior Mutability 与 Inherited Mutability 是对应关系, 一个变量是否可变, 取决于它的使用环境, 而不是类型就称为承袭可变性.

在 Rust 中, 常见的具备内部可变性的类型有 Cell, RefCell, Mutex, RwLock, Atomic 等.

## 2	Cell

### 2.1	Description

```rust
use std::cell::Cell;

fn main() {
    let data: Cell<i32> = Cell::new(100);
    let p = &data;
    data.set(20);
    println!("{}", p.get());

    p.set(39);
    println!("{}", p.get());
}
```

上面的 data 变量并没有使用 mut 关键字修饰, 但我们仍然实现了对其内部值的编辑, 这就是所谓的内部可变性.

这种方式似乎破坏了 Rust 的原则, 但实际上, 这种类型是完全符合内存安全的, Rust 要尽力避免 Alias 和 Mutation 同时存在的原因是防止在一个 mut 指针修改内存的过程中, 数据以一种被破坏的状态被其他的指针观测到.

Cell 类型泽不会出现这种情况, 因为 Cell 将数据包裹在内部, 用户无法获取指向其内部数据的指针.
### 2.2	API

```rust
impl<T> Cell<T> {
	
	pub fn get_mut(&mut self) -> &mut T { }

	pub fn set(&self, val: T) { }

	pub fn swap(&self, other: &Self) { }
	
	pub fn replace(&self, val: T) -> T { }

	pub fn into_inner(self) -> T { }
 
}
```

