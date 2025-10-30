---
type: Rust
sub-type: Smart Pointers
---
`Drop` Trait 解决了手动资源管理容易出错的问题, 如忘记释放, 多次释放等, 提供了自动的, 确定性的资源清理机制.

## 1	The What

```rust
trait Drop {
	fn drop(&mut self);
}
```

在 `drop` 中编写清理逻辑, 比如 `file.close()` `socket.shutdown()` 等, 入参为可变引用, 可以修改自身, `drop` 方法的真正执行者是编译器, 编译器会在变量离开作用域时, 自动插入调用 `drop` 的代码.

## 2	The Mechanism

- **绑定作用域:** `drop` 总是在变量离开其作用域 (scope) 的 `{}` 处被调用.
    
- **LIFO 原则:** 在同一个作用域内, 变量按其创建顺序的 **相反** 顺序被 drop (后进先出).
    
- **递归销毁:**
    
    - 当一个 struct 被 drop 时, _首先_ 执行它自己实现的 `drop` 方法.
        
    - _然后_, 按字段在 struct 中声明的顺序, 依次 drop 它的每一个字段.

## 3	The Rules

1. `drop` 方法不允许被使用调用, 容易与编译器的调用产生冲突, 如果必须要提前 `drop` 掉, 可以使用 `std::mem::drop(my_value)` 方法, 这个函数会获取所有权并立即将其清除;
	
2. `Drop` Trait 和 `Copy` Trait 是互斥的, Copy 在语义上的浅拷贝的按位复制, 如果一个实现了 Drop 的类型可以被 Copy, 那副本和原始值就会指向同一个资源, 导致重复释放.
	
3.  当一个线程发生 Panic 时, Rust 会开始栈回溯, 在回溯过程中, 会依次调用路径上所有变量的 `drop` 方法.