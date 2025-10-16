---
type: Rust
sub-type: Error Handling
---
## 1	Definition

Result 用于处理可恢复的错误, 它的定义为:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

Rust 中的 `std::fs::File::open` 函数返回的是 `Result`: 

```rust
let hello = File::open("hello.txt");
if let Result::Ok(file)  = hello {
	println!("Successfully opened the file: {:?}", file);
} else {
	panic!("File name is not exist");
}
```

## 2	Matching on Different Errors

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let greeting_file_result = File::open("hello.txt");

    let greeting_file = match greeting_file_result {
        Ok(file) => file,
        Err(error) => match error.kind() {
            ErrorKind::NotFound => match File::create("hello.txt") {
                Ok(fc) => fc,
                Err(e) => panic!("Problem creating the file: {e:?}"),
            },
            _ => {
                panic!("Problem opening the file: {error:?}");
            }
        },
    };
}
```

我们可以通过调用 io 库 Error 的 kind 方法来获取其具体的错误类型, 如文件未找到, 没有该目录等, 并可以对这些不同种类的错误设置单独的处理方案.

### 2.1	Alternatives to Using match with Result<T, E>

That’s a lot of `match`! The `match` expression is very useful but also very much a primitive. In Chapter 13, you’ll learn about closures, which are used with many of the methods defined on `Result<T, E>`. These methods can be more concise than using `match` when handling `Result<T, E>` values in your code.

For example, here’s another way to write the same logic as shown in Listing 9-5, this time using closures and the `unwrap_or_else` method:

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let greeting_file = File::open("hello.txt").unwrap_or_else(|error| {
        if error.kind() == ErrorKind::NotFound {
            File::create("hello.txt").unwrap_or_else(|error| {
                panic!("Problem creating the file: {error:?}");
            })
        } else {
            panic!("Problem opening the file: {error:?}");
        }
    });
}
```

Although this code has the same behavior as Listing 9-5, it doesn’t contain any `match` expressions and is cleaner to read. Come back to this example after you’ve read Chapter 13, and look up the `unwrap_or_else` method in the standard library documentation. Many more of these methods can clean up huge nested `match` expressions when you’re dealing with errors.

### 2.2	Shortcuts for Panic on Error: unwrap and expect

`Result` 中一个 `unwrap` 方法, 用于解包 `Ok` 返回其内部的值, 如果对一个 `Err` 执行 `unwrap`, 会触发 panic:

```rust
use std::fs::File;

fn main() {
    let greeting_file = File::open("hello.txt").unwrap();
}
```

---

`except` 方法能够让你指定触发 panic 时的报错信息, 一般在生产中, 会选择这个方法来解包.

```rust
use std::fs::File;

fn main() {
    let greeting_file = File::open("hello.txt")
        .expect("hello.txt should be included in this project");
}
```

## 3	Propagating Errors

### 3.1	How to Propagate Errors?

书写的函数中出现错误的时候, 不必急于立刻处理, 可以选择将错误打包返回给拥有更多 context 的调用者, 这称为 Propagation.

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let username_file_result = File::open("hello.txt");

    let mut username_file = match username_file_result {
        Ok(file) => file,
        Err(e) => return Err(e),
    };

    let mut username = String::new();

    match username_file.read_to_string(&mut username) {
        Ok(_) => Ok(username),
        Err(e) => Err(e),
    }
}
```

比如上面封装的一个 username 阅读函数, 将错误返回.

### 3.2	A Shortcut for Propagating Errors: The `?` Operator

来看一个极端的 Propagation Errors 案例:

```rust
match open_file() {
    Ok(file) => {
        match read_contents(file) {
            Ok(contents) => {
                match parse_contents(contents) {
                    Ok(number) => {
                        // 终于成功了!
                        Ok(number) 
                    },
                    Err(e) => Err(e), // 第三步失败, 返回错误
                }
            },
            Err(e) => Err(e), // 第二步失败, 返回错误
        }
    },
    Err(e) => Err(e), // 第一步失败, 返回错误
}
```

这种代码结构被称为 Pyramid of Doom, 它及其冗长且难以阅读.

`?` 操作符在一定程度上可以解决这样的痛点, 它的核心使命是将这种 "如果是 `OK`  取出值继续, 如果是 `Err` 立即返回错误" 的语义简化, 压缩成一个单一的符号, 比如上面的案例, 使用 `?` 操作符可以改写为:

```rust
fn simplified_process_data() -> Result<i32, ErrorType> {
    // 1. 尝试打开文件。如果失败，立即返回 Err(io::Error 转换为 ErrorType)
    let file = open_file()?; 
    
    // 2. 尝试读取内容。如果失败，立即返回 Err(io::Error 转换为 ErrorType)
    let contents = read_contents(file)?; 
    
    // 3. 尝试解析内容。如果失败，立即返回 Err(parse_error 转换为 ErrorType)
    let number = parse_contents(contents)?; 
    
    // 4. 所有步骤都成功，返回 Ok(值)
    Ok(number)
}
```

### 3.3	Principle of ? Operator

Rust 的 `?` Operator 实际上是对下面这个 `match` 语句的 Syntactic Sugar:

```rust
match expression {
    Ok(value) => value,      // 如果结果是 Ok, 整个 `expression?` 就等于里面包含的 value.
    Err(error) => return Err(error.into()), // 如果是 Err, 所在的函数立刻返回一个 Err.
}
```

成功时, unwrap 这个 Ok, 取出内部的值作为表达式的值;

失败时, 会出发一个 Early Return, 不会继续执行内层的代码, 而是操控外层函数直接返回; 其中的 `into` 方法, 会将当前的错误转化成外部函数返回值中声明的错误类型, 这使得你可以在一个函数中组合使用返回多个不同错误类型的函数, 只要其能够转化为相同的错误类型即可.

这个错误类型的转化能力被定义在 From Trait 中, 为了让 `?` 操作符能够在函数中顺利工作, 该函数返回的错误类型 `E'` 必须实现从被调用函数返回的错误类型到自身的 `From<E>` Trait, 如:

```rust
// 必须实现 From Trait 才能让 ? 成功转换
impl From<std::io::Error> for CustomError {
    fn from(err: std::io::Error) -> CustomError {
        // 转换逻辑，例如将 IO 错误包装成 CustomError::IO
        CustomError::IO(err)
    }
}
```