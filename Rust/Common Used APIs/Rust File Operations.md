---
type: Rust
sub-type: APIs
---
`std::fs` 模块提供了一系列跨平台的文件系统操作 API, 提供了对文件或目录的读, 写, 删除能力.

## 1	Common File APIs

`File::create(path)`: 创建一个新文件, 如果已存在则会覆盖;

`File::open(path)`: 打开一个已存在的文件 for reading;

`fs::read_to_string(path)`: 将文件中的内容读入一个 `String`;

`fs::write(path, content)`: 向文件中写入内容, 会覆盖先前的内容;

`OpenOptions::new().append(true).open(path)`: 以 append 模式打开一个文件;

`fs::copy(src, dst`: 将文件从 src 拷贝到 dst;

`fs::rename(src, dst)`: 相当于 mv 指令, 移动文件或重命名;

`fs::remote_file(path)`: 删除文件.

## 2	Common Directory APIs

`fs::create_dir(path)`: 创建一个新的目录;

`fs::create_dir_all(path)`: 递归的创建一个目录和它的父目录;

`fs::read_dir(path)`: 读取目录内容, 返回一个迭代器;

`fs::remove_dir(path)`: 删除一个空目录;

`fs::remove_dir_all`: 递归的删除一个目录及其所有的内容.

## 3	Metadata and Path Handling

`fs::metadata(path)`: 返回文件的元数据, 如大小, 所有权, 时间戳等;

`fs::canonicalize(path)`: 返回一个绝对的, 规范化的路径;

`DirEntry::path()`: 返回一个目录的完整路径.