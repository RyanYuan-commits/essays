---
type: Rust
sub-type: Basic
---
## 1	The Tuple Type

### 1.1	Brief Introduction

元组是一个将多个不同类型的值组合进一个复合类型的主要方式. 元组长度固定, 一旦声明, 其长度不会增大或缩小.

```rust
fn main() {
    let tup: (i32, f64, u8) = (500, 6.4, 1);

    // 使用模式匹配解构元组
    let (x, y, z) = tup;
    println!("The value of y is: {}", y);

    // 使用点号 . 访问元组元素
    let five_hundred = tup.0;
    println!("The value of tup.0 is: {}", five_hundred);
}
```

### 1.2	The Unit Type

元组中可以一个元素也没有, 这样的元素有一个单独的名字, 叫做 unit 单元类型, 当表达式没有任何返回时, 会默认返回 unit.

```rust
let empty: () = ();
```

空元组和空结构体占用的内存空间为 0:

```rust
struct Foo {}

fn type_of<T>(_: &T) -> &'static str {
    std::any::type_name::<T>()
}

fn main() {
    let empty_struct = Foo{};
    let unit1 = ();
    let unit2 = {};

    println!("{}", type_of(&empty_struct)); // demo_projects::Foo
    println!("{}", type_of(&unit1)); // ()

    println!("{}", std::mem::size_of_val(&empty_struct)); // 0
    println!("{}", std::mem::size_of_val(&unit1)); // 0
    println!("{}", std::mem::size_of_val(&unit2)); // 0
}
```

## 2	[[Example Programs#1 Example Programs Using Strucs|The Struct Type]]

结构体 (struct) 是一种自定义数据类型, 允许你将多个相关的值组合成一个有意义的组合, 与元组不同, 每个元素都有自己的名字;

```rust
struct User {
    active: bool,
    username: String,
    email: String,
    sign_in_count: u64,
}

fn main() {
    let user1 = User {
        email: String::from("someone@example.com"),
        username: String::from("someusername123"),
        active: true,
        sign_in_count: 1,
    };

    println!("Username: {}", user1.username);
}
```

### 2.1	Using the Field Init Shorthand

当初始化时传入的变量名和 struct 的属性名相同时, 可以不需要重复:

```rust
fn build_user(email: String, username: String) -> User {
    User {
        active: true,
        username,
        email,
        sign_in_count: 1,
    }
}
```

### 2.2	Creating Instances from Other Instances

```rust
fn main() {
    let user1 = User {
        active: true,
        username: String::from("RyanYuan"),
        email: String::from("ryan@123.com"),
    };

    let user2 = User {
        active: false,
        ..user1
    };

    // println!("{}", user1.username); // Error: value borrowed here after move
}

// ====== 等同于 ======

let user2 = User {
	active: false,
	username: user1.username,
	email: user1.email,
};
```

上面的写法等同于将 user2 未设值的属性赋成 user1 的属性, 如果属性未实现 `Copy` trait, 执行的是所有权转移.

### 2.3	tuple struct

元组结构体 (tuple struct) 是一种类似元组的结构体, **但它有自己的类型名**.

```rust
struct Color(i32, i32, i32);
struct Point(i32, i32, i32);

fn main() {
    let black = Color(0, 0, 0);
    let origin = Point(0, 0, 0);

    println!("Color: ({}, {}, {})", black.0, black.1, black.2);
    println!("Point: ({}, {}, {})", origin.0, origin.1, origin.2);
}
```

### 2.4	[[Tuple & Struct#1.2 The Unit Type|Unit]]-Like Structs Without Any Fields

Unit-List Structs 指的是没有任何属性的 Struct, 它的表现类似 unit 类型; 主要用于希望赋予一些 trait 给某些类型, 但是这个类型并没有需要存储的变量;

```rust
struct UnitStruct;

fn main() {
    // Create a instance.
    let u = UnitStruct;
}
```

## 3	The Enum Type

枚举 (enum) 允许你定义一个可以枚举其所有可能成员的类型.

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}

fn main() {
    let msg = Message::Write(String::from("hello"));

    match msg {
        Message::Quit => {
            println!("The Quit variant has no data to destructure.");
        }
        Message::Move { x, y } => {
            println!(
                "Move in the x direction {} and y direction {}",
                x, y
            );
        }
        Message::Write(text) => println!("Text message: {}", text),
        Message::ChangeColor(r, g, b) => {
            println!(
                "Change the color to red {}, green {}, and blue {}",
                r, g, b
            )
        }
    }
}
```
