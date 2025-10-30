---
type: Rust
sub-type: Smart Pointers
---
`Rc<T>` 解决了 Rust 严格的 "单一所有权" 规则无法满足 "共享所有权" 需求的问题, 当一份数据, 如 Graph 的节点, 或者共享的配置, 需要被多个所有者拥有, 且在编译时无法确定谁是最后一个所有者(无法确定销毁时机), 就需要使用 `Rc<T>` 它是一种 **Runtime** 的所有权管理机制.

## 1	The What

当调用 `Rc::new(value)` 方法后, 会在堆上分配一块内存, 包含数据本身和控制块, 控制块中含有:

- Strong Count: 强引用, 记录 `Rc<T>` 指针的数量;

- Weak Count: 记录 `Weak<T>` 指针的数量, 用于解决循环引用.

使用 `Rc::new` 来创建数据, Strong Count = 1, 后续可以使用 `Rc::Clone` 来将 Strong Count + 1, 当引用离开其作用域时, Strong Count 会自动减一, 若减到 0, 则立即 `drop` 并释放堆内存. 

## 2	The Rules

### 2.1	Single-Threading vs Multi-Threading

`Rc<T>` (Reference Counted): **只用于单线程**. 它的计数增减是非原子的, 速度快.
    
`Arc<T>` (Atomic Reference Counted): **用于多线程**. 它的计数是原子操作, 保证线程安全, 性能略低.
    
试图在线程间传递 `Rc<T>` 会导致编译错误.

### 2.2	Immutability

`Rc<T>` 默认只提供对数据的 **不可变** 引用, 如果数据被多方共享, 如果一方能任意修改, 会对其他方造成混乱.

如果想要修改其中的数据, 可以结合 `RefCell` 或者 `Mutex` 使用.

### 2.3	Reference Cycles

在 Rust 中, 如果两个对象互相持有对方的 Reference Counting, 则两者的计数器永远不会为 0, 此时就出现了循环引用问题;

这时候需要使用 Weak Reference 来解决, `Weak<T>` 是一种特殊的智能指针, 使用 `Rc::downgrade(rc_ptr)` 来创建.

