---
type: Rust
sub-type: Functional Programming
---
迭代器模式实现了在条目序列上, 依次执行某个任务, 我们只需要指定任务, 而不需要去手动的获取每个条目或判断迭代何时结束.

## 1	Iterator are Lazy

在 Rust 中, Iterator 是惰性的, 只有当它被消费时才会真正的开始工作, 当调用 `vec.iter()`, `map()` 或 `filter()` 等方法的时候, 这些方法不会立刻执行, 而是会构造一个迭代器链, 描述 " 如何处理逻辑 ", 而不是立即执行.

```rust
let v = vec![1, 2, 3];
let iter = v.iter().map(|x| x + 1);
```

上面的代码并不会做任何事, 仅当调用诸如 `collect()`, `sum()`, `for_each()`, `next()` 等消费者方法时, 迭代才真正的发生.

## 2	The Iterator Trait and the `next` Method

所有迭代器都实现了标准库定义的 `Iterator` Trait, 该 Trait 的定义类似这样:

```rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;

    // 这里省略了有着默认实现的方法
}
```

`Iterator` Trait 只需要实现者, 定义一个 `next` 方法, 该方法会一次返回一个封装在 `Some` 中的迭代器条目, 当迭代完毕时, 就会返回 `None`.

```rust
#[test]
fn iterator_demonstration() {
    let v1 = vec! [1, 2, 3];

    let mut v1_iter = v1.iter();

    assert_eq! (v1_iter.next(), Some(&1));
    assert_eq! (v1_iter.next(), Some(&2));
    assert_eq! (v1_iter.next(), Some(&3));
    assert_eq! (v1_iter.next(), None);
}
```

手动调用迭代器的方法时, 需要将迭代器构造为可变, 因为调用 `next` 方法时, 会修改迭代器用来追踪其位于序列中何处的内部状态. 每次对 `next` 的调用, 都会消耗掉迭代器的一个条目, 在使用 `for` 循环时, 之所以不需要将 `v1_iter` 构造为可变, 是由于那个循环取得了 `v1_iter` 的所有权, 而在幕后将其构造为了可变.

## 3	Iterator Methods

### 3.1	Methods that Consume the Iterator

调用 `next` 的方法称为 Consuming Adaptor, 因为它们会耗尽迭代器, 比如 `sum` 方法, 它会获取迭代器的所有权并重复调用 `next` 方法来迭代条目, 从而消费迭代器;

```rust
#[test]
fn iterator_sum() {
    let v1 = vec! [1, 2, 3];

    let v1_iter = v1.iter();

    let total: i32 = v1_iter.sum();

    assert_eq! (total, 6);
}
```

由于 `sum` 取得了迭代器的所有权, 因此, 在该方法调用后, 就无法再使用迭代器了.

### 3.2	Iterators tht Produce Other Iterators

Iterator Adaptor 是定义在 `Iterator` Trait 上, 不会消费迭代器的方法, 相反, 这些方法会通过改变初始迭代器的某一方面, 从而产生出另一个迭代器;

```rust
let v1 = vec! [1, 2, 3];

let v2: Vec<_> = v1.iter().map(|x| x + 1).collect();

assert_eq! (v2, vec! [2, 3, 4]);
```