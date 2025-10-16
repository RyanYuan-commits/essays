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

