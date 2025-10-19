---
type: Rust
sub-type: Basic
created: 2025-10-05 15:58:31
updated: 2025-10-05 20:06:57
---
## 1	Functions

### 1.1	A Brief Desc

Rest 的函数使用关键字 `fn` 开头, 函数可以有一系列的输入函数, 还可以有一个返回类型, 函数体包含一系列语句或表达式;

函数返回可以使用 return 语句, 也可以使用表达式, 下面的案例中使用到了[[Pattern Destructure]]

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

### 1.2	The Diverging Function

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

### 1.3	The Main Function

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

### 1.4	The Const Function

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

### 1.5	Recursive Call of Function

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

## 2	Method Syntax

Rust 中的 Method 与 Function 一样, 使用 `fn` 关键字来定义, 并且都有参数和返回值, 调用方式也类似... 与函数不同的是, Method 需要定义在 [[Tuple & Struct#2 The Struct Type|Struct]], [[Enums and PatternMatching|Enum]] or [[Trait]] 下, 并且 Method 的第一个参数永远是 `self`, 表示的是调用这个方法的实例.

### 2.1	Defining Methods

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn calcute_area(&self) -> u32 {
       self.height * self.width 
    }
}

fn main() {
    let rect1 = Rectangle {
        width: 30,
        height: 50,
    };
    
    println!(
        "The area of the rectangle is {} square pixel",
        rect1.calcute_area()
    )
}
```

成员方法在 impl 块中定义, 调用时, 可以使用 `instance_name.method_name` 的方式来调用. impl 块除了定义成员方法还可以定义静态函数, 实现 Trait, 实现泛型约束.

### 2.2	Self

方法 `calcute_area` 的入参是 `&self`, 它实际上是 `self: &Self` 的缩写, 在 impl block 中, `Self` is an alias for the type that `impl` block is for; 成员方法的第一个参数必须是 Self 类型或包装在 Self 之上的类型:

```rust
trait T1 {
    fn method1(self: Self);
    fn method2(self: &Self);
    fn method3(self: &mut Self);
}

// === 两种写法是相同的 ===

trait T2 {
    fn method1(self);
    fn method2(&self);
    fn method3(&mut self);
}
```