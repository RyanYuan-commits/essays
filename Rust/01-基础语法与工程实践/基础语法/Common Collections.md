---
type: Rust
sub-type: Basic
---
## 1	Storing Lists of Values with [Vectors](https://doc.rust-lang.org/std/vec/struct.Vec.html)

### 1.1	Creating a New Vector

```rust
let v: Vec<i32> = Vec::new();

let v = vec![1, 2, 3];
```

Vector 存储的是相同类型的变量, 类型通过泛型指定或编译器自动推断.

### 1.2	Updating a Vector

```rust
fn main() {
    let mut v: Vec<i32> = Vec::new();
    v.push(123);
    v.push(21);

    println!("{:?}", v); // Output: [123, 21]
}
```

如果想要修改 Vector 中的内容, 需要将其置为 mutable, Rust 可以通过 push 方法传入的类型来自动推导 Vector 的类型.

### 1.3	Reading Elements of Vectors

```rust
let v = vec![1, 2, 3, 4, 5];

let third: &i32 = &v[2];
println!("The third element is {third}");

let third: Option<&i32> = v.get(2);
match third {
    Some(third) => println!("The third element is {third}"),
    None => println!("There is no third element."),
}
```

Rust 提供了两种方式来获取 Vector 中的值, 通过 get 返回的是一个 Option, 在越界或者其他情况下更加安全.

另外, 当外部持有 Vector 的借用时, 就需要遵守 Rust 的借用规则, 下面的这个代码是不被允许的:

```rust
fn main() {
    let mut v = vec![1, 2, 3, 4, 5];

    let first = &v[0];

    v.push(6);

    println!("The first element is: {first}");
}
```

### 1.4	Iterating Over the Values in a Vector

通过迭代器来遍历 Rust 的 Vector, 如果要修改, 通过 `&mut` 借用:

```rust
fn main() {
    let mut v = vec![1, 2, 3, 4, 5];
    
    for i in &v {
        println!("{}", i);
    }

    for i in &mut v {
        *i = *i + 1;
    }
}
```

## 2	Storing UTF-8 Encoded Text with Strings

### 2.1	What Is a String?

Rust 的 core language 只有一种 string 类型, 就是 string **slice** `str`, 字符串字面量也是 string slice.

String 是 Rust 标准库中提供的类型, 是可增长, 可编辑, 且拥有所有权的 string **type**.

### 2.2	Creating a New String

String 实质上是对 Vector of Bytes 的一个包装, 提供更多的保障, 限制以及能力, 所以大部分适用于 Vector 的方法同样适用于 String, 比如, 创建一个 String:

```rust
let mut s = String::new();
```

对于实现了 Display Trait 的类型, 可以通过 `to_string()` 来基于此类型创建一个 String, 这种做法等效于 `String::from`;

```rust
let s = "123";
let string = s.to_string();

// ====== Same ======

let s = String::from("initial contents");
```

### 2.3	Updating a String

除了类似 Vector 的修改能力以外, String 还可以使用 `+` 或者 `format!` 来拼接 String values: 

```rust
// Example 1: 使用 `+` 运算符拼接 String
fn main() {
    let s1 = String::from("Hello"); // 创建一个 String
    let s2 = " World"; // 创建一个字符串切片 (&str)

    // 使用 `+` 拼接 String 和 &str
    // 注意：s1 在拼接后会被移动 (所有权转移)，不能再使用
    let s3 = s1 + s2; 
    println!("拼接结果 (使用 +): {}", s3); // 打印结果
}

// Example 2: 使用 `format!` 宏拼接 String
fn main() {
    let part1 = String::from("Rust"); // 创建一个 String
    let part2 = " programming"; // 创建一个字符串切片 (&str)
    let part3 = " is fun!"; // 另一个字符串切片

    // 使用 `format!` 宏拼接多个值，结果是新的 String
    // `format!` 不会获取参数的所有权，所以 part1 仍然可用
    let full_string = format!("{}{}{}", part1, part2, part3);
    println!("拼接结果 (使用 format!): {}", full_string); // 打印结果
    
    // part1 依然可用
    println!("原始 String 依然可用: {}", part1);
}
```

### 2.4	Indexing into Strings

Rust 中, 不允许直接通过下标来取用 String 中的字符, 因为 String 在内部是以 UTF-8 编码的字节序列（`Vec<u8>`）存储的. 

一个 Unicode 字符在 UTF-8 编码下可能占据一到四个字节. 因此, 一个整数下标并不总能对应到一个完整的、有效的 Unicode 字符边界, 直接索引字节可能导致获取到不完整的字符编码, 这会破坏 String 的有效性并引入难以预料的错误, 比如

```rust
let s = String::from("你好world");

// 尝试直接通过整数下标访问字符 (编译错误).
let first_char = s[0]; // 错误: the type `String` cannot be indexed by `{integer}`.
```

如果一定要输出每个位置的字符, 可以通过 `bytes` 方法:

```rust
fn main() {
    let s = String::from("你好world");

    for (i, _) in s.bytes().enumerate() {
        print!("{} ", i); // Output: 0 1 2 3 4 5 6 7 8 9 10
    }
}
```

---

Rust 字符串切片必须符合 UTF-8 规范. 因此, Slice 操作无法创建不符合 UTF-8 规范的切片.

```rust
// ====== String Slice ======
let valid_slice = &s[0..3];
println!("{}", valid_slice); // Output: 你

let invalid_slice = &s[0..1]; // Panic: byte index 1 is not a char boundary; it is inside '你' (bytes 0..3) of `你好world`

// ====== 正确的字符迭代方式 ======

println!("通过.chars()迭代字符:");
for (i, c) in s.chars().enumerate() {
    print!("字符[{}] = '{}' ", i, c);
}
println!("");
```

## 3	Storing Keys with Associated Values in HashMaps

### 3.1	Creating a New Hash Map

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();

scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);
```

使用 `new` 方法创建, HashMap 的 K, V 必须是相同类型的.

### 3.2	Accessing Values in a Hash Map

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();

scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);

let team_name = String::from("Blue");
let score = scores.get(&team_name).copied().unwrap_or(0);
```

默认的 `get` 方法返回的是 Option 实例, 上面的代码通过 Option 提供的方法返回的是拷贝内容.

HashMap 支持迭代访问:

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();

scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);

for (key, value) in &scores {
    println!("{key}: {value}");
}
```

## 4	Hash Maps and Ownership

对于实现了 Copy Trait 的类型, Hash Maps 会进行值拷贝, 而对于其他类型, Hash Maps 会获得其所有权:

```rust
use std::collections::HashMap;

let field_name = String::from("Favorite color");
let field_value = String::from("Blue");

let mut map = HashMap::new();
map.insert(field_name, field_value);
// field_name and field_value are invalid at this point, try using them and
// see what compiler error you get!
```

### 4.1	Updating a Hash Map

```rust
// ====== 覆盖旧的 value ======
use std::collections::HashMap;

let mut scores = HashMap::new();

scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Blue"), 25);

println!("{scores:?}");

// ====== 只在 key 不存在时插入 ======
let mut scores = HashMap::new();
scores.insert(String::from("Blue"), 10);

scores.entry(String::from("Yellow")).or_insert(50);
scores.entry(String::from("Blue")).or_insert(50);

println!("{scores:?}");


// 基于旧值修改
let text = "hello world wonderful world";
let mut map = HashMap::new();

for word in text.split_whitespace() {
    let count = map.entry(word).or_insert(0);
    *count += 1;
}

println!("{:?}", map);
```