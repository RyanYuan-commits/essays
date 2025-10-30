---
type: Rust
sub-type: Smart Pointers
---
`Rc<T>` 解决了 Rust 严格的 "单一所有权" 规则无法满足 "共享所有权" 需求的问题, 当一份数据, 如 Graph 的节点, 或者共享的配置, 需要被多个所有者拥有, 且在编译时无法确定谁是最后一个所有者(无法确定销毁时机), 就需要使用 `Rc<T>` 它是一种 **Runtime** 的所有权管理机制.

## 1	The What

当调用 `Rc::new(value)` 方法后, 会在堆上分配一块内存, 包含数据本身和控制块, 控制块中含有:

- Strong Count: 强引用, 记录 `Rc<T>` 指针的数量;

- Weak Count: 记录 `Weak<T>` 指针的数量, 用于解决循环引用.

使用 `Rc::new` 来创建数据, Strong Count = 1, 后续可以使用 `Rc::Clone` 来将 Strong Count + 1,   