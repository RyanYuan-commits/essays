---
type: Rust
sub-type: Type System
---
## 1	The Why

如果需要编写一个函数, 功能是在一个列表中找出最大的那个元素, 如果没有泛型, 那那你需要定义一组函数, 会导致大量的代码重复:

```rust
find_largest_i32(list: &[i32]) -> i32
find_largest_f64(list: &[f64]) -> f64  
find_largest_char(list: &[char]) -> char
```

Generic Type 可以解决这种冗余的问题, 它允许我们在定义函数时, 使用抽象的占位符类型, 而不必真正的指定具体类型.

## 2	The What

### 2.1	Core Components

在函数, 结构体, 或者枚举名称后面, 用 `<>` 包裹一个占位符名称来声明一个泛型, 如 `<T>`:

```rust
fn my_func<T>(arg: T) { ... }

struct Point<T> { x: T, y: T }

enum Option<T> { Some(T), None }
```

声明完成后, 就可以在 Scope 中像使用真实类型一样去使用泛型参数.

还可以对泛型进行 Trait 限制, 具体的语法为: `<T: TraitName>`, 可以确保输入的类型一定是实现了这个 Trait 的.

### 2.2	How it Works

Rust 的泛型通过一个叫做单态化的过程来实现, 当你使用不同的类型去调用泛型函数时, 编译器会分析这些调用, 并在最终的编译产物中, 为每一种不同的调用都生成一个特化的, 非泛型的实现. 所以, Rust 的泛型在运行是运行时零成本的, 最终生成的机器码和手动编写重复代码一样快.



