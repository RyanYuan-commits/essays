### 1 什么是 RDB？
关于 RDB(Redis Database) 文件，官网是这样介绍的：
```
RDB (Redis Database): RDB persistence performs point-in-time snapshots of your dataset at specified intervals.
```
RDB 持久化的是按 **指定的时间间隔执行** 数据集的时间点 **快照**。
RDB 快照是默认开启的，默认情况下，会在满足以下条件的时候触发，下面的内容摘自 `redis.conf`
```
# Unless specified otherwise, by default Redis will save the DB:
#   * After 3600 seconds (an hour) if at least 1 change was performed
#   * After 300 seconds (5 minutes) if at least 100 changes were performed
#   * After 60 seconds if at least 10000 changes were performed
#
# You can set these explicitly by uncommenting the following line.
#
# save 3600 1 300 100 60 10000
```
- 一小时内发生了 1 个变化
- 五分钟内发生了 100 个变化
- 一分钟内发生了 10000 个变化
在我们运行 Redis 的时候，也可以使用 `SAVE` 或 `BGSAVE` 命令来手动的保存 RDB 快照。
其中，`SAVE` 指令是同步的（**synchronous**），也就是执行 `SAVE` 的时候，会阻塞其他客户端的请求，所以官方是推荐我们使用 `BGSAVE` 来进行快照的存储的，但是在一些 Redis 无法 fork 出子进程的时候，比如系统资源不足、系统调用错误等情况，还是需要执行 `SAVE` 方法。
```
You almost never want to call `SAVE` in production environments where it will block all the other clients. Instead usually BGSAVE is used. However in case of issues preventing Redis to create the background saving child (for instance errors in the fork(2) system call), the `SAVE` command can be a good last resort to perform the dump of the latest dataset.
```
`BGSAVE` 中的 `BG` 就是 Background 的意思，它会 fork 出一个子进程来执行 RDB 快照的存储，具体的实现方式放到后面来说明。
虽然配置文件中对于 RDB 的时间间隔配置使用的是 `save`，但实际上自动行的是 `BGSAVE` 指令，原因同上。
### 2 优缺点
#### 2.1 RDB 的优点
##### 方便的时间点备份
```
RDB is a very compact single-file point-in-time representation of your Redis data. RDB files are perfect for backups. For instance you may want to archive your RDB files every hour for the latest 24 hours, and to save an RDB snapshot every day for 30 days. This allows you to easily restore different versions of the data set in case of disasters.
```
RDB 是 Redis 数据的非常紧凑的单文件时间点表示形式。RDB 文件非常适合备份。例如，您可能希望在最近 24 小时内每小时存档一次 RDB 文件，并在 30 天内每天保存一个 RDB 快照。这使您可以在发生灾难时轻松恢复不同版本的数据集。
但是需要注意，Redis 存储 RDB 快照的时候，为了安全性的考虑，采用的是先创建一个新的 RDB 文件，然后当写入完成之后，使用新的文件去覆盖旧的 RDB 文件，因此，如果需要备份的话，可以采取这些措施：
**修改文件名**：在生成 RDB 文件时，可以通过修改 Redis 配置文件中的 dbfilename 选项为一个动态命名的文件（例如使用时间戳），这样每次快照都会生成一个新文件而不是覆盖原文件。
- 例如，可以在配置文件中设置为：`dbfilename dump-$(date +%Y%m%d%H%M%S).rdb`。不过这种方法需要在 Redis 启动时就进行设置。
**使用外部工具**：定期备份 RDB 文件到其他位置，比如使用定时任务将 dump.rdb 文件复制到其他目录，并为文件添加时间戳。
##### 快速的灾难恢复和启动
RDB 的紧凑性，为它带来了这样的好处：
```
RDB is very good for disaster recovery, being a single compact file that can be transferred to far data centers, or onto Amazon S3 (possibly encrypted).
```
作为一个紧凑的文件，RDB 非常适合灾难恢复，可以传输到远数据中心或 Amazon S3（一种对象存储服务）。
```
RDB allows faster restarts with big datasets compared to AOF.
```
与 AOF 相比，RDB 允许更快地重新启动大型数据集。
#### 2.2 RDB 的缺点
因为 RDB 的存储并不是实时进行的，所以，它无法保证在断电之后的完全恢复，只能恢复到已经存储快照的那个时间点上；
如果需要在 Redis 停止工作时（例如在停电后）最大限度地减少数据丢失的可能性，那 RDB 是无法做到的，因此，如果 Redis 因任何原因停止工作而没有正确关闭，最近几分钟的数据就会丢失。
虽然 RDB 使用了子进程做到了异步存储，但是每次创建 RDB 快照都需要 fork 一个子进程（fork 是同步的，可能会阻塞其他客户端），相比于 AOF，RDB 执行 fork 的频率显然是非常高的。而如果数据集非常大，或者服务器的 CPU 性能不是很强，fork 一个子进程可能会耽误几毫秒的时间。如果确实是服务器 CPU 性能较差，或者数据集过大，可以稍微降低一下生成 RDB 的频率，使用 AOF，因为 AOF 虽然也需要 fork，但是频率相较于 RDB 是低很多的。
### 3 RDB 实现原理
在执行 BGSAVE 命令的时候，Redis 是可以同时为其他客户端提供服务的，关键的就在于 **写时复制** 技术。
执行 bgsave 命令的时候，会通过 `fork()` 创建子进程，此时子进程和父进程是共享同一片内存数据的，因为创建子进程的时候，会复制父进程的页表，但是页表指向的物理内存还是一个。

![[写时复制刚 fork 的时候.png|500]]
只有在发生修改内存数据的情况时，物理内存才会被复制一份。
![[写时复制第一次修改.png|500]]
这样的目的是为了减少创建子进程时的性能损耗，从而加快创建子进程的速度，毕竟创建子进程的过程中，是会阻塞主线程的。
所以，创建 bgsave 子进程后，由于共享父进程的所有内存数据，于是就可以直接读取主线程（父进程）里的内存数据，并将数据写入到 RDB 文件。当主线程（父进程）对这些共享的内存数据也都是只读操作，那么，主线程（父进程）和 bgsave 子进程相互不影响。
但是，如果主线程（父进程）要修改共享数据里的某一块数据（比如键值对 A）时，就会发生写时复制，于是这块数据的物理内存就会被复制一份（键值对 A），然后主线程在这个数据副本（键值对 A）进行修改操作。与此同时，bgsave 子进程可以继续把原来的数据（键值对 A）写入到 RDB 文件。
就是这样，Redis 使用 bgsave 对当前内存中的所有数据做快照，这个操作是由 bgsave 子进程在后台完成的，执行时不会阻塞主线程，这就使得主线程同时可以修改数据。在最极端情况下，如果所有的共享内存都被修改，则此时的内存占用是原先的 2 倍，所以，针对写操作多的场景，我们要留意下快照过程中内存的变化，防止内存被占满了。