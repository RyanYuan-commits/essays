---
type: Rust
sub-type: Concurrent Programming
---
编程语言实现线程的方式各不相同, 许多操作系统都提供了提供给语言调用以创建新线程的 API, Rust 标准库中的线程与系统线程是一对一的关系.

## 1	Creating a New Thread with `spawn`

Rust 通过 `thread::spawn` 函数来创建线程, 传递参数是一个 [[Closures]], 并在其中包含了希望新线程运行的代码:

```rust
use std::{thread, time::Duration};

fn main() {
    thread::spawn(|| {
        for i in 1..10 {
            println!("{:?}", i);
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("{:?}", i);
        thread::sleep(Duration::from_millis(1));
    }
}
```

上面的函数中, 主线程的子线程分别打印数字 1 到 5, 但是当主线程结束时, 无论是否完成, 子线程都会一起结束, 运行可以观察到子线程打印结果是不完整的.

## 2	Wait for All Threads to Finish

上面的代码由于线程结束, 子线程中的代码没有被完全执行, 而在一些情况下, 因为无法保证线程运行的顺序, 我们甚至无法保证线程会被执行.

线程创建方法 `thread::spawn` 有一个 `JoinHandle` 类型的返回值, 可以通过调用 `join` 方法来等待线程执行结束, join 方法返回一个 Result 对象, 成功结果中封装的是一个空元组.

