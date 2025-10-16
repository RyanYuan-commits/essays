---
type: Rust
sub-type: Basic
---
## 1	Immutability

Rust 中, 变量使用 `let` 关键字声明:

```rust
fn main() {
    let x = 5;
    println!("x 的值为: {}", x);
    
    // 下面这行代码会报错，因为 x 是不可变的
    x = 6; 
}
```

使用 `let` 声明的变量是不可变的, 这意味着, 一旦给一个变量赋值, 就不能再改变它的值, 在多线程环境下, 声明为 `let` 的变量可以被安全读取, 无需担心数据竞争.

## 2	Mutability

如果想修改一个变量的值, 需要使用 `mut` 关键字来使其变为可变的:

```rust
fn main() {
    // 使用 mut 关键字声明可变变量
    let mut y = 5;
    println!("y 的初始值为: {}", y);

    // 现在可以改变 y 的值了
    y = 6;
    println!("y 改变后的值为: {}", y);
}
```

## 3	Shadowing

Rust 允许你使用相同的变量名来声明一个新变量, 这个新变量会完全遮蔽掉之前的同名变量, 提供了较好的命名便利;

```rust
fn main() {
    let z = 5;
    println!("z 的初始值为: {}", z);

    // 遮蔽之前的 z
    let z = z + 1;
    println!("z 遮蔽后的值为: {}", z);

    {
        // 内部作用域的遮蔽
        let z = z * 2;
        println!("内部作用域中 z 的值为: {}", z);
    } // 内部作用域结束，z 恢复为外部的值

    println!("外部作用域中 z 的值为: {}", z);
}
```

## 4	Type Alias

可以使用 `type` 关键字给同一个类型起一个 type alias:

```rust
type Age = u32;

fn grow(age: Age, year:u32) -> Age {
	age + year
}
```

类型别名还可以用在泛型场景:

```rust
type Double<T> = (T, Vec<T>)
```

在后续使用 `Double<i32>` 时, 就等同于 `(i32, Vec<i32>)`, 可以简化代码.

## 5	Static Variables

使用 `static` 关键字声明静态变量, Rust 没有为静态变量提供类型推断能力, 所以 `static` 变量必须显示的指定类型: 

```rust
static MAX_POINTS: u32 = 100_000;

fn main() {
    let a: &f32;
    {
        static PI: f32 = 3.14;
        a = &PI;
    }
    println!("{:?}", *a); // Output: 3.14
}
```

静态变量的生命周期不受当前作用域的制约, 均为  `'static`, 从程序启动到结束.