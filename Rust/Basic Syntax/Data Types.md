---
type: Rust
sub-type: Basic
---
## 1	bool

布尔类型表示 "是" 或 "否" 的二值逻辑, 有两个值 `true` 和 `false`, 一般用在逻辑表达式中, 可以执行 "与或非" 等运算;

```rust
fn main() { 
    let x = true; 
    let y: bool = !x;

    let z = x && y; // 逻辑与
    println!("{}", z); 

    let z = x || y; // 逻辑或
    println!("{}", z); 

    let z = x & y; // 按位与
    println!("{}", z); 

    let z = x | y; // 按位或
    println!("{}", z); 
    
    let z = x ^ y; // 按位异或
    println!("{}", z);
}
```

bool 类型的表达式可以用在 `if/while` 等表达式中, 作为条件表达式.

```rust
if a >= b {
	...
} else {
	...
}
```


## 2	Integer

Rust 提供了多种整型, 用于表示整数, 分为有符号 (signed) 和无符号 (unsigned) 两种, 数字后缀可以用来显式指定类型, 例如 `57u8`.

- **有符号整型**: 以 `i` 开头, 可以表示正数, 负数和零. (例如 `i8`, `i32`, `i64`)
	
- **无符号整型**: 以 `u` 开头, 只能表示非负数. (例如 `u8`, `u32`, `u64`)

| **长度**      | **有符号**     | **无符号**     |
| ------- | ------- | ------- |
| 8-bit   | `i8`    | `u8`    |
| 16-bit  | `i16`   | `u16`   |
| 32-bit  | `i32`   | `u32`   |
| 64-bit  | `i64`   | `u64`   |
| 128-bit | `i128`  | `u128`  |
| arch    | `isize` | `usize` |

`isize` 和 `usize` 的长度取决于程序运行的计算机体系结构 (64 位架构上是 64 位, 32 位架构上是 32 位).

```rust
fn main() {
    let a: i32 = -10; // 有符号 32 位整数
    let b: u64 = 100; // 无符号 64 位整数
    
    // 使用类型后缀
    let c = 57u8;
    
    println!("a = {}, b = {}, c = {}", a, b, c);
}
```

## 3	Floating-Point

Rust 有两种浮点数类型, `f32` 和 `f64`, 分别表示单精度和双精度浮点数.

- `f32`: 32 位单精度浮点数.    
	
- `f64`: 64 位双精度浮点数, 是默认类型.

```rust
fn main() {
    let x = 2.0; // 默认是 f64
    let y: f32 = 3.0; // 显式指定为 f32

    println!("x = {}, y = {}", x, y);
}
```

## 4	Character

Rust 的 `char` 类型用于表示单个 **Unicode** 字符, 大小为 4 个字节, 字符字面量使用单引号 `'` 括起来.

```rust
fn main() {
    let c = 'z';
    let z = 'ℤ';
    let heart_eyed_cat = '😻';

    println!("{}, {}, {}", c, z, heart_eyed_cat);
}
```