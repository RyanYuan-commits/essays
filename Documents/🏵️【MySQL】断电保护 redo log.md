### 1 为什么需要 redo log？
`Buffer Pool` 提高了读写效率，但它是基于内存的，而内存总是不可靠，万一断电重启，还没来得及落盘的脏页数据就会丢失。
为了防止断电导致数据丢失，当有一条记录需要更新的时候，`InnoDB` 引擎会执行这两步操作：
1. 先更新 `Buffer Pool`（同时标记为脏页）
2. 然后将本次对这个页的修改以 `redo log` 的形式记录下来，这个时候更新就算完成了。
![[undo 日志.png|600]]

`redo log` 是物理日志，记录了某个数据页做了什么修改，比如对 XXX 表空间中的 YYY 数据页 ZZZ 偏移量的地方做了 AAA 更新，每当执行一个事务就会产生这样的一条或者多条物理日志，在事务提交时，会先将 `redo log` 持久化到磁盘。
当系统崩溃时，虽然 `Buffer Pool` 中的脏页数据没有被持久化，但是 `redo log` 已经持久化，接着 MySQL 重启后，可以根据 `redo log` 的内容，将所有数据恢复到最新的状态。
### 2 写入原理
#### 2.1 顺序写
`redo log` 采用的是顺序写，执行的是追加操作，而写入数据执行的是随机写，需要先找到写入位置，然后才写入磁盘。
磁盘的「顺序写」比「随机写」 高效的多，因此 `redo log` 写入磁盘的开销更小。
至此， 针对为什么需要 redo log 这个问题我们有两个答案：
- 实现事务的持久性，让 MySQL 有 `crash-safe` 的能力，能够保证 MySQL 在任何时间段突然崩溃，重启后之前已提交的记录都不会丢失；
- 将写操作从「随机写」变成了「顺序写」，提升 MySQL 写入磁盘的性能。
#### 2.2 缓冲机制
 执行一个事务的过程中，产生的 `redo log` 也不是直接写入磁盘的，因为这样会产生大量的 I/O 操作，而且磁盘的运行速度远慢于内存。
 所以，`redo log` 也有自己的缓存，`redo log buffer`，每当产生一条 `redo log` 时，会先写入到缓存区，后续再持久化到磁盘：
 ![[read log buffer.png|600]]
` redo log buffer` 默认大小 16 MB，可以通过 `innodb_log_Buffer_size` 参数动态的调整大小，增大它的大小可以让 MySQL 处理「大事务」是不必写入磁盘，进而提升写 IO 性能。
#### 2.3 刷盘时机
缓存在 redo log buffer 里的 redo log 还是在内存中，其刷入磁盘的时机主要有：
- `MySQL` 正常关闭时；
- 当 `redo log buffer` 中记录的写入量大于 `redo log buffer` 内存空间的一半时，会触发落盘；
- `InnoDB` 的后台线程每隔 1 秒，将 `redo log buffer` 持久化到磁盘。
- 每次事务提交时都将缓存在 `redo log buffer` 里的 `redo log` 直接持久化到磁盘。
#### 2.4 循环写
默认情况下， InnoDB 存储引擎有 1 个重做日志文件组，由有 2 个 redo log 文件组成，名为 `ib_logfile0` 和 `ib_logfile1` 。
在重做日志组中，每个日志文件的大小是固定且一致的，假设每个日志文件设置的上限是 1 GB，那么总共就可以记录 2GB 的操作。
重做日志文件组是以**循环写**的方式工作的，从头开始写，写到末尾就又回到开头，相当于一个环形。InnoDB 存储引擎会先写 `ib_logfile0` 文件，当 `ib_logfile0` 文件被写满的时候，会切换至 `ib_logfile1` 文件，当 ib_logfile1 文件也被写满时，会切换回 ib_logfile0 文件。
我们知道 redo log 是为了防止 Buffer Pool 中的脏页丢失而设计的，那么如果随着系统运行，Buffer Pool 的脏页刷新到了磁盘中，那么 redo log 对应的记录也就没用了，这时候我们擦除这些旧记录，以腾出空间记录新的更新操作。
redo log 是循环写的方式，相当于一个环形，InnoDB 用 write pos 表示 redo log 当前记录写到的位置，用 checkpoint 表示当前要擦除的位置：
![[read log 循环写.png|600]]
如果 write pos 追上了 checkpoint，就意味着redo log 文件满了，这时 MySQL 不能再执行新的更新操作，也就是说 MySQL 会被阻塞（因此所以针对并发量大的系统，适当设置 redo log 的文件大小非常重要），此时会停下来将 Buffer Pool 中的脏页刷新到磁盘中，然后标记 redo log 哪些记录可以被擦除，接着对旧的 redo log 记录进行擦除，等擦除完旧记录腾出了空间，checkpoint 就会往后移动（图中顺时针），然后 MySQL 恢复正常运行，继续执行新的更新操作。
