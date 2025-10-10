---
type: Rust
sub-type: Ownership
---
一个变量的生命周期就是从它创建到销毁的整个过程:

```rust
fn main() {
    let v: i32 = 10; // lifetime of v starts here
    {
        let x = v + 1; // lifetime of x starts here
        println!("{}", x); // lifetime of x is ended here
    }
    println!("{}", v); // lifetime of v is ended here
}
```

