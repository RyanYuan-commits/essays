---
type: Rust
sub-type: Basic
created: 2025-10-05 15:58:31
updated: 2025-10-05 20:06:57
---
## 1 简介

Rest 的函数使用关键字 `fn` 开头, 函数可以有一系列的输入函数, 还可以有一个返回类型, 函数体包含一系列语句或表达式;

函数返回可以使用 return 语句, 也可以使用表达式, 下面的案例中使用到了[[模式解构]]

```rust
fn add((x, y): (i32, i32)) -> i32 {
	t.0 + t.1
}
```

函数也可以不写返回类型, 在这种情况下, 编译器会认为返回类型是 `()`;

函数可以当成头等公民 (first class value) 被赋值给某个变量, 这个值可以像函数一样被调用:

```rust
fn add((x, y) : (i32, i32)) -> i32 {
    x + y
}

fn main() {
    let fun = add;
    let result = fun((2, 3));
    println!("Result: {}", result); // Output: Result: 5
}
```

## 2 发散函数

Rust 支持一种特殊的发散函数 (Diverging functions), 它的返回类型是 `!`:

```rust
fun diverges() -> ! {
	panic!("This function never returns!");
}
```

因为 `panic!` 会直接导致栈展开, 它的返回类型就是一个特殊的 `!` 符号, 这种函数叫做发散函数;

发散类型的最大特点是, 它可以被转化为任意一个类型:

```rust
let x: String = diverges();
let y: i32 = diverges();
```

## 3 main 函数

Rust 中 main 函数的参数传递和返回状态码都由单独的 API 来完成:

```rust
fn main() {
    for arg in std::env::args() {
        println!("{}", arg);
    }
    std::process::exit(0);
}
```

进程可以在任何时候调用 `exit` 直接退出, 退出时错误码在参数中指定;

如果要读取环境变量, 可以用 `std::env::var` 以及 `std::env::vars` 函数获取:

```rust
fn main() {
    for var in std::env::vars() {
        println!("{:?} : {:?}", var.0, var.1);
    }
}
```

## 4 const fun

`const fn` 用于定义可以在编译时求值的函数. 这意味着, `const fn` 的返回值可以用于常量表达式中, 例如数组的长度, `static` 变量的初始化等.

```rust
const fn add(a: i32, b: i32) -> i32 {
    a + b
}

const SUM: i32 = add(1, 2);

fn main() {
    println!("SUM is {}", SUM); // 编译时就已经计算出结果 3
}
```

`const fn` 内部有一些限制:

- 不能包含不安全的 `unsafe` 代码.
- 不能使用浮点数运算.
- 不能分配堆内存.

## 5 函数递归调用

函数可以调用自身, 这就是递归. 递归函数需要一个明确的终止条件, 否则会无限递归下去, 直到栈溢出.

下面是一个计算阶乘的递归函数示例:

```rust
fn factorial(n: u64) -> u64 {
    if n == 0 {
        1
    } else {
        n * factorial(n - 1)
    }
}

fn main() {
    println!("Factorial of 5 is {}", factorial(5)); // 120
}
```

虽然 Rust 支持递归, 但它没有对尾递归进行优化 (Tail Call Optimization). 这意味着, 如果递归深度过大, 可能会导致栈溢出. 在这种情况下, 最好使用迭代的方式来代替递归.