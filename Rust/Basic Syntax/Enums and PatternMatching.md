---
type: Rust
sub-type: Basic
---
## 1	Defining an Enum

Enum 用于解决: 如何创建一个自定义类型, 该类型的值被限制在一个有限可能性的集合中.

### 1.1	Enum Values

```rust
enum IpAddrKind {
    V4(u8, u8, u8, u8),
    V6(String),
}

fn main() {
    let four = IpAddrKind::V4(127, 0, 0, 1);
    let six = IpAddrKind::V6(String::from("::1"));
}
```

`Enum` 也可以定义方法:

```rust
impl Message {
	fn call(&self) {
		// method body would be defined here
	}
}

let m = Message::Write(String::from("hello"));
m.call();
```

### 1.2	[The Option Enum](https://doc.rust-lang.org/std/option/enum.Option.html)

在 Rust 中, `Option<T>` 是一个泛型 enum, 它解决了我们想要一种方法来表示一个值可能缺失, 同时又能在编译时就杜绝所有因空值引发的运行时崩溃.

```rust
enum Option<T> {
    Some(T), // 变体 `Some`，表示存在一个类型为 T 的值。
    None,    // 变体 `None`，表示不存在值。
}
```

`Option<T>` 和 `T` 是两种完全不同的, 不兼容的两种类型, 不能直接将两者混淆使用, 你必须 "打开" 这个 Option, Rust 提供了多种安全的方式来做这件事:

```rust
fn main() {
    let maybe_number: Option<i32> = Some(42);

    // 方式1: if let
    if let Some(n) = maybe_number {
        println!("if let: The number is: {}", n);
    } else {
        println!("if let: No number found.");
    }

    // 方式2: match
    match maybe_number {
        Some(n) => println!("match: The number is: {}", n),
        None => println!("match: No number found."),
    }

    // 方式3: unwrap
    // 注意: 如果为 None 会 panic
    println!("unwrap: The number is: {}", maybe_number.unwrap());

    // 方式4: unwrap_or
    println!("unwrap_or: The number is: {}", maybe_number.unwrap_or(0));

    // 方式5: map
    maybe_number.map(|n| println!("map: The number is: {}", n));

    // 方式6: as_ref + map
    maybe_number.as_ref().map(|n| println!("as_ref + map: The number is: {}", n));

    // 方式7: expect
    println!("expect: The number is: {}", maybe_number.expect("No number found"));
}
```

## 2	The match Control Flow Construct

### 2.1	Syntax of match

```rust
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter,
}

fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => {
            println!("Lucky penny!");
            1
        }
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
    }
}
```

### 2.2	Patterns That Bind to Values

通过模式匹配, match 可以让我们将元素内部的某些值绑定在变量上.

```rust
#[derive(Debug)]
enum UsState {
    Alabama,
    Alaska
}

enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter(UsState)
}

fn main() {
    let quarter = Coin::Quarter(UsState::Alabama);

    let res: u8 = match quarter {
        Coin::Dime => 1,
        Coin::Nickel => 5,
        Coin::Penny => 10,
        Coin::Quarter(state) => {
            println!("State quarter from {state:?}!");
            25
        }
    };
}
```

### 2.3	Matches Are Exhaustive

Rust match 的分支必须包含所有的可能性, 否则是无法通过编译的. 这一严格的规则是Rust安全哲学的重要体现, 它强制开发者在编译时就考虑所有可能的输入状态. 这种设计杜绝了运行时出现未处理情况的风险, 从而避免了潜在的bug和程序崩溃;

然而, 这并不意味着开发者必须为每一个枚举变体都编写独立的逻辑. Rust 提供了 `_` 或者 `other` 关键字来表示其他情况:

```rust
let dice_roll = 9;
match dice_roll {
	3 => add_fancy_hat(),
	7 => remove_fancy_hat(),
	other => move_player(other),
}

fn add_fancy_hat() {}
fn remove_fancy_hat() {}
fn move_player(num_spaces: u8) {}
```

此外, 对于只需要匹配单个模式的场景, `if let` 语句提供了一种更简洁的替代方案, 避免了完整 `match` 的冗余,

## 3	Concise Control Flow with if let and let else

`if let` 语法允许你在只需要匹配固定的一个分支时简化语法:

```rust
let mut count = 0;
match coin {
	Coin::Quarter(state) => println!("State quarter from {state:?}!"),
	_ => count += 1,
}

// === Same ===

let mut count = 0;
if let Coin::Quarter(state) = coin {
	println!("State quarter from {state:?}!");
} else {
	count += 1;
}
```

可以使用 let else 语法简化 if let 赋值:

```rust
fn describe_state_quarter(coin: Coin) -> Option<String> {
    let state = if let Coin::Quarter(state) = coin {
        state
    } else {
        return None;
    };

    if state.existed_in(1900) {
        Some(format!("{state:?} is pretty old, for America!"))
    } else {
        Some(format!("{state:?} is relatively new."))
    }
}

// ====== Same ======

fn describe_state_quarter(coin: Coin) -> Option<String> {
    let Coin::Quarter(state) = coin else {
        return None;
    };

    if state.existed_in(1900) {
        Some(format!("{state:?} is pretty old, for America!"))
    } else {
        Some(format!("{state:?} is relatively new."))
    }
}

```
