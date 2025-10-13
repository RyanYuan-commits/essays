---
type: Rust
sub-type: ProjectManagement
---
## 1	Packages

A Cargo feature that lets you build, test, and share crates;

## 2	Crate

Crate 是 Rust 编译器认为的最小代码组织单位, 即使是使用 `rustc` 来编译单文件, 编译器也会将其认定为一个 Creat; Create 可以包含 Modules, 这些 Modules 可能会被定义在其他的文件中, 这些文件会与该 Crate 一起编译.

Crate 有两种类型, Binary Crate or Library Crate:

- Binary Crate: 可以被编译为可执行文件, 例如命令行工具或服务, Binary Crate 中必须包含一个 main 函数;
- Library Crates: 不包含 Main 函数, 也不会被编译成可执行文件, 其作用是定义在各个项目下使用的功能.

每一个 Package 可以包含多个 Binary Crate, 但是只能包含一个 Library Crate.
