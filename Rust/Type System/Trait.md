---
type: Rust
sub-type: Basic
created: 2025-10-05 15:58:34
updated: 2025-10-07 10:28:04
---
## 1 Method

```rust
trait Shape {
	fn area(&self) -> f64;
}
```

所有 trait 类型都有一个隐藏的类型 `Self`, 代表当前这个实现了此 trait 的具体类型;

trait 中定义的函数, 也可以称作关联函数;

函数第一个参数如果是 Self 相关的类型, 且命名为 `self`, 这个参数可以被称为 "receiver", 具有 "receiver" 的函数, 我们称之为方法, 可以通过变量实例使用小数点来调用;

没有 receiver 的函数, 我们成为静态函数, 可以通过类型 + `::` 的方式来调用, 在 Rust 中, 函数和方法没有本质区别.

---

在 Rust 中 `Self` 和 `self` 都是关键字, `Self` 是类型名, `self` 是变量名;

`self` 参数同样可以指定类型, 但是必须是包装在 `Self` 类型之上的类型:

```rust
trait T1 {
    fn method1(self: Self);
    fn method2(self: &Self);
    fn method3(self: &mut Self);
}

// 上下两种写法是完全一样的

trait T2 {
    fn method1(self);
    fn method2(&self);
    fn method3(&mut self);
}
```

---

impl trait 案例:

```rust
trait Shape {
    fn area(&self) -> f64;
}

struct Circle {
    radius: f64,
}

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
}

fn main() {
    let circle = Circle { radius: 5.0 };
    println!("The area of the circle is: {}", circle.area());
}
```

另外, 针对一个类型, 我们可以直接对它 impl 来增加成员方法, 无需 trait 名字:

```rust
impl Circle {
	fn get_radius(&self) -> f64 {
		self.radius
	}
}
```

可以将这段代码看作是 Circle 类型实现了一个匿名的 trait, 用这种方式定义的方法叫做这个类型的内在方法 inherent methods.

---

trait 中可以包含方法的默认实现, 如果方法在 trait 中已经有了方法体, 那么在实现的时候, 可以选择不用重写;

比如, 在标准库中, `Iterator` 这个 trait 就包含了十多种方法, 其中只有 `fn next(&mut self) -> Option<Self::Item>` 是没有默认实现的, 其他方法均有默认实现, 在实现迭代器的时候只需要挑选需要重写的方法即可.

## 2 Static Methods

没有 receiver 参数的方法, 称为静态方法, 可以通过 `Type::FunctionName` 的方式调用; 需要注意的是, 即使第一个参数是 Self 相关类型, 只要变量名字不是 Self, 也会被识别为静态方法.

```rust
trait Bird {
    fn area() -> f64;
}

struct Sparrow;

impl Bird for Sparrow {
    fn area() -> f64 {
        0.5 * 0.3
    }
}

fn main() {
    let area = Sparrow::area();
    println!("Area of Sparrow: {}", area);
}
```

## 3 拓展方法

我们还可以利用 trait 给其他类型添加成员方法, 哪怕这个类型不是由我们实现的:

```rust
trait Double {
    fn double(&self) -> Self;
}

impl Double for i32 {
    fn double(&self) -> i32 {
        *self * 2
    }
}

fn main() {
    let x = 5;
    println!("Double of {} is {}", x, x.double());
}
```

Rust 对拓展方法做了一些限制, 在声明 trait 和实现 trait 的时候, Rust 规定了一个 Coherence Rule: trait 和类型两者之中至少有一个是在当前的 crate 定义的.

## 4 完整的函数调用语法

Rust 提供了一种统一的函数调用语法, 称为 "Universal Function Call Syntax" (UFCS).

这种语法允许我们明确指定要调用哪个 trait 的方法, 尤其是在多个 trait 具有同名方法时非常有用.

```rust
trait Pilot {
    fn fly(&self);
}

trait Wizard {
    fn fly(&self);
}

struct Human;

impl Pilot for Human {
    fn fly(&self) {
        println!("This is your captain speaking.");
    }
}

impl Wizard for Human {
    fn fly(&self) {
        println!("Up!");
    }
}

fn main() {
    let person = Human;
    
    Pilot::fly(&person);
    
    // Type as trait 方式
    <Human as Wizard>::fly(&person);
}
```

语法格式为 `<Type as Trait>::function(&self)`.

## 5 trait 约束和继承

### 5.1 约束

Trait 约束 (Trait Bounds) 用于限制泛型参数必须实现特定的 trait.

```rust
use std::fmt::Display;

fn print_something<T: Display>(item: T) {
    println!("{}", item);
}

fn main() {
    print_something(5);
    print_something("hello");
}
```

上面的 `T: Display` 就是一个 trait 约束, 它表示 `T` 类型必须实现 `Display` trait.

我们也可以使用 `where` 子句来指定 trait 约束, 这在有多个泛型参数时更清晰:

```rust
use std::fmt::Debug;

fn print_debug<T, U>(t: T, u: U)
where
    T: Display + Debug,
    U: Clone + Debug,
{
    // ...
}
```

### 5.2 继承

一个 trait 可以继承自另一个 trait. 这意味着, 如果你想实现子 trait, 你必须先实现父 trait.

```rust
trait Person {
    fn name(&self) -> String;
}

// Student trait 继承自 Person trait
trait Student: Person {
    fn university(&self) -> String;
}

struct John;

impl Person for John {
    fn name(&self) -> String {
        "John".to_string()
    }
}

impl Student for John {
    fn university(&self) -> String {
        "MIT".to_string()
    }
}

fn main() {
    let john = John;
    println!("Name: {}, University: {}", john.name(), john.university());
}
```

在这个例子中, `Student` trait 继承了 `Person` trait. 因此, 任何实现 `Student` trait 的类型也必须实现 `Person` trait.

## 6 Derive

Rust 提供了一个 `#[derive]` 属性, 可以让我们很方便地为自定义类型实现一些常用的 trait, 例如 `Debug`, `Clone`, `Copy` 等.

```rust
#[derive(Debug, Clone, Copy)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = p1; // p1 实现了 Copy trait, 所以这里是拷贝而不是移动
    
    println!("{:?}", p2); // p2 实现了 Debug trait, 所以可以被打印
}
```

`derive` 实际上是一种过程宏, 编译器会根据 `derive` 的 trait, 自动为我们生成相应的 `impl` 代码块.

## 7 别名

我们可以使用 `type` 关键字为 trait 创建别名, 尤其是在 trait 约束很长的情况下, 可以让代码更简洁.

```rust
use std::fmt::Debug;

// 为 `Debug + Clone` 创建一个别名
trait MyTrait: Debug + Clone {}
impl<T: Debug + Clone> MyTrait for T {} // 实现了 Debug + Clone 认为其实现了 MyTrait, 与前一个语句绑定

fn my_function<T: MyTrait>(item: T) {
    println!("{:?}", item.clone());
}

fn main() {
    my_function(5);
}
```

从 Rust 1.27 版本开始, 我们可以直接使用 `type` 关键字为 trait 创建别名:

```rust
type MyTrait = std::fmt::Debug + Clone;

fn my_function<T: MyTrait>(item: T) {
    println!("{:?}", item.clone());
}

fn main() {
    my_function(5);
}
```

这种方式更加简洁直观.

## 8 标准库中常见的 trait

### 8.1 Display 和 Debug

```rust
// std::fmt::Debug
pub trait Display: PointeeSized {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result;
}

// std::fmt::Display
pub trait Debug: PointeeSized {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result;
}
```

它们的主要作用在格式化输出时 (如 `println!`):

- 只有实现了 `Display` 的类型, 才能用 `{}` 格式实现打印;
- 只有实现了 `Debug` 的类型, 才能用 `{:?} {:#?}` 的格式打印.

### 8.2 PartialOrd/Ord & PartialEq/Eq

在 Rust 中, 因为 NaN 的存在, 浮点数的不具备全序关系的, 这意味着传统的比较运算符如 `<`, `>`, 在遇到 `NaN` 值时会表现出非直观的行为. 

这种非直观行为源于 `NaN` 的特殊性质: `NaN != NaN` 总是成立, 并且任何涉及 `NaN` 的比较操作 (例如 `NaN < X` 或 `NaN > X`) 都会返回 `false`. 

因此, Rust 为浮点数实现了 `PartialOrd` 特性, 而非 `Ord`. 

`PartialOrd` 允许比较结果为 `Option<Ordering>`, 当 `NaN` 参与比较时返回 `None`, 精确反映了这种部分排序关系.

---

同样地, `PartialEq` 和 `Eq` 特性也受到 `NaN` 的影响. 由于 `NaN != NaN` 的事实, 浮点数只实现了 `PartialEq`, 而未能实现 `Eq`. 

`Eq` 要求比较操作具有自反性 (reflexivity), 即 `a == a` 必须始终为 `true`, 而 `NaN` 显然违反了这一原则. 这意味着你不能直接将浮点数作为 `BTreeSet` 或 `HashMap` 的键值, 因为这些数据结构通常要求其键类型实现 `Ord` 和 `Eq` 特性.

### 8.3 Sized

Sized trait 是 Rust 中一个非常重要的 trait, 它的定义如下:

```rust
#[doc(alias = "?", alias = "?Sized")]
#[stable(feature = "rust1", since = "1.0.0")]
#[lang = "sized"]
#[diagnostic::on_unimplemented(
    message = "the size for values of type `{Self}` cannot be known at compilation time",
    label = "doesn't have a size known at compile-time"
)]
#[fundamental]
#[rustc_specialization_trait]
#[rustc_deny_explicit_impl]
#[rustc_do_not_implement_via_object]
#[rustc_coinductive]
pub trait Sized: MetaSized {
    // Empty.
}
```

这个 trait 没有任何的成员方法, 它有 `#[lang="sized"]` 属性, 说明它与普通的 trait 不同, 编译器对它有特殊的处理;

用户也不能针对自己的类型实现这个 trait, 一个类型是否满足 `Sized` 约束的编译器推导的, 用户无权指定;

### 8.4 Default

Rust 中没有构造函数的概念, 它只提供了类似 Java 的静态工厂方法. 

在 Rust 中, 我们通常使用关联函数(Associated Functions)来创建类型实例. 这些函数并不像传统构造函数那样直接在对象创建时被调用, 而是作为普通函数显式地返回一个新实例. 习惯上, 这种用于初始化的关联函数会被命名为 `new`.

这种设计提供了更大的灵活性, 允许一个类型拥有多个不同的初始化方法, 例如 `new` 用于标准创建, `from_string` 用于从字符串解析, 或 `default` 用于提供默认值. 更重要的是, 关联函数可以返回 `Result<T, E>` 类型, 这意味着它们可以在创建失败时返回一个错误, 而不是通过异常处理, 这与 Rust 的错误处理哲学完美契合.