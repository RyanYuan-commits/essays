### 1 什么是 bin log？
前面介绍的 undo log 和 redo log 这两个日志都是 Innodb 存储引擎生成的。
MySQL 在完成一条更新操作后，Server 层还会生成一条 binlog，等之后事务提交的时候，会将该事物执行过程中产生的所有 binlog 统一写入 binlog 文件。
binlog 文件是记录了所有数据库表结构变更和表数据修改的日志，不会记录查询类的操作，比如 SELECT 和 SHOW 操作。
### 2 作用
这个问题跟 MySQL 的时间线有关系。
最开始 MySQL 里并没有 `InnoDB` 引擎，MySQL 自带的引擎是 `MyISAM`，但是 `MyISAM` 没有 `crash-safe` 的能力，`bin log` 日志只能用于归档。
而 `InnoDB` 是另一个公司以插件形式引入 MySQL 的，既然只依靠 `bin log` 是没有 `crash-safe`（崩溃恢复）能力的，所以 `InnoDB` 使用 redo log 来实现 `crash-safe` 能力。

关于 `bin log`  和 `redo log` 的区别，来思考这样一个问题：**如果不小心将整个数据库的文件全部删除，使用 redo 文件能够恢复数据吗？**
答案是不可以使用 `redo log` 文件恢复，只能使用 `binlog` 文件恢复。
- 因为 `redo log` 文件是循环写，只记录未被刷入磁盘的数据的物理日志，已经刷入磁盘的数据都会从 `redo log` 文件里擦除。
- `binlog` 文件保存的是全量的日志，也就是保存了所有数据变更的情况，理论上只要记录在 `binlog` 上的数据，都可以恢复，所以如果不小心整个数据库的数据被删除了，得用 `binlog` 文件恢复数据。