---
type: Rust
---
## 1	Example Programs Using Strucs

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let rect1 = Rectangle {
        width: 30,
        height: 50,
    };
    
    println!(
        "The area of the rectangle is {} square pixel",
        calcute_area(&rect1)
    )
}

fn calcute_area(&Rectangle{height, width}: &Rectangle) -> u32 {
    height * width
}
```

