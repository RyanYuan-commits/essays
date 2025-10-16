---
type: Rust
sub-type: ProjectManagement
---
## 1	How Compiler works?

**从 Crate 根文件开始**: 编译器从根文件开始查找需要编译的内容. 根文件通常是 `src/lib.rs` 或 `src/main.rs`.

**声明模块**: 在根文件中, 你可以声明模块. 例如, 当你使用 `mod garden` 声明一个名为 `garden` 的模块时, 编译器会从以下位置查找其代码:

*   `mod garden` 声明后紧跟的大括号内的代码.
*   文件 `src/garden.rs`.
*   文件 `src/garden/mod.rs`.

**声明子模块**: 在非根文件中的模块内, 可以定义子模块. 例如, 在 `src/garden.rs` 文件中添加 `mod vegetables;`, 编译器会从以下位置查找相关代码:

*   紧跟在 `mod vegetables` 声明后的花括号内代码.
*   文件 `src/garden/vegetables.rs`.
*   文件 `src/garden/vegetables/mod.rs`.

**模块内代码的路径**: 一旦模块被 Crate 包含, 就可以在该 Crate 的任何位置使用它. 例如, 如果 `Asparagus` 结构体定义在 `garden::vegetables` 模块中, 可以通过 `crate::garden::vegetables::Asparagus` 来引用它.

```rust
// ====== src/garden/mod.rs ======
pub mod vegetables;


// ====== src/garden/vegetables.rs ======
#[derive(Debug)]
pub struct Asparagus;


// ====== src/main.rs ======
mod garden;

use crate::garden::vegetables::Asparagus;

fn main() {
    let asparagus = Asparagus;
    println!("{:?}", asparagus); // Output: Asparagus
}
```

**私有与公开**: 子模块默认为私有 (Private), 除其父模块外, 其他位置无法直接访问. 需要使用 `pub mod xxx` 将其声明为公开 (Public) 模块.

## 2	Grouping Related Code in Modules

```rust
// src/lib.rs
mod fron_of_hourse {
    mod hosting {
        fn add_to_waitlist() {}

        fn seat_at_table() {}
    }

    mod serving {
        fn take_order() {}

        fn take_payment() {}
    }
}
```

其模块树为:

```plaintext
crate
 └── front_of_house
     ├── hosting
     │   ├── add_to_waitlist
     │   └── seat_at_table
     └── serving
         ├── take_order
         ├── serve_order
         └── take_payment
```

## 3	Path for Referring to an Item in the Module Tree

要在模块中定位到我们要使用的内容, 需要使用到路径, 路径分为相对路径和绝对路径, 路径中的节点使用 `::` 分割;

```rust
mod fron_of_hourse {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
	// 绝对路径
    crate::fron_of_hourse::hosting::add_to_waitlist();
    // 相对路径
    fron_of_hourse::hosting::add_to_waitlist();
}
```

可以通过 `super` 关键字指定父模块中的 items:

```rust
mod fron_of_hourse {

    fn deliver_order() {}

    pub mod hosting {
        pub fn add_to_waitlist() {
            super::deliver_order();
            cook_order();

        }

        fn cook_order() {}
    }
    
}
```

## 4	The use Keyword

### 4.1	Bring Path into Scope with the use Keyword

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
}
```

通过上面的方式, 就好像将 hosting 定义在当前模块一样, 但是这种定义方式只在当前模块生效, 子模块无法直接使用:

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use crate::front_of_house::hosting;

mod customer {
    pub fn eat_at_restaurant() {
        hosting::add_to_waitlist();
    }
}
```

上面的代码编译会报错; 可以将  use 语句移到子模块中, 或者使用 super::hosting 来引用父模块中的快捷方式.

### 4.2	Providing New Name with the as Keyword

可以通过 as 关键字来为引入的路径取一个别名:

```rust
use std::fmt::Result;
use std::io::Result as IoResult;

fn function1() -> Result {
    // --snip--
}

fn function2() -> IoResult<()> {
    // --snip--
}
```

### 4.3	Re-exporting Names whith pub use

当我们使用 `use` 关键字将名称引入作用域时, 该名称在我们导入的作用域内是是有的, 为了让作用域之外的代码能够使用这个名称, 可以使用 `pub use` 关键字:

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
}
```

在此更改之前, 外部代码需要通过路径 `restaurant::front_of_house::hosting::add_to_waitlist` 调用 `add_to_waitlist` 函数, 这还需要将 `front_of_house` 模块标记为 `pub`. 现在这个 `crate` 已经从根模块重新导出了 `front_of_house` 模块, 外部代码可以使用路径 `restaurant::hosting::add_to_waitlist` 替代.

### 4.4	Using External Packages

```rust
// Cargo.toml
[dependencies]
rand = "0.8.5"

// src/main.rs
use rand::Rng;

fn main() {
    let r = rand::thread_rng().gen_range(1..=100);
    println!("{:?}", r);
}
```

标准 `std` 库也是一个位于我们包外部的 Crate, 由于标准库是随 Rust 语言一同发布的, 我们不需要修改 Cargo.toml 来包含 `std`, 但是我们仍然需要通过 `use` 关键字来引用这个标准库, 来将它的类目引入我们包的命名空间, 例如, 对于 `HashMap`, 我们会使用这样的语句:

```rust
use std::collections::HashMap;
```

### 4.5	Using Nested Paths to Clean Up Large use Lists

```rust
use std::cmp::Ordering;
use std::io;

// ====== Same ======

use std::{cmp::Ordering, io};
```

对于两条 use 路径, 其中一条是另一条的子路径的情况:

```rust
use std::io;
use std::io::Write;

// ====== Same ======

use std::io::{self, Write};
```

### 4.6	The Glob Operator

若要将路径中定义的所有公共项引入作用域, 可以在路径后跟 `*` 通配符运算符:

```rust
use std::collections::*;
```

这个 `use` 语句会将 `std::collections` 中定义的所有公共项引入当前作用域;

使用通配符时要格外小心, 通配符可能导致难以分辨作用域中存在哪些名称, 以及程序中使用的名称是在何处定义的.

此外, 如果依赖项更改了其定义, 你导入的内容也会随之改变, 这可能在升级依赖项时引发编译器错误——例如当依赖项添加了与你在同一作用域中的定义同名的定义时.