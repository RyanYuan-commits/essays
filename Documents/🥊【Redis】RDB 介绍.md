# 什么是 RDB？
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
# 优缺点
## RDB 的优点
### 方便的时间点备份
```
RDB is a very compact single-file point-in-time representation of your Redis data. RDB files are perfect for backups. For instance you may want to archive your RDB files every hour for the latest 24 hours, and to save an RDB snapshot every day for 30 days. This allows you to easily restore different versions of the data set in case of disasters.
```
RDB 是 Redis 数据的非常紧凑的单文件时间点表示形式。RDB 文件非常适合备份。例如，您可能希望在最近 24 小时内每小时存档一次 RDB 文件，并在 30 天内每天保存一个 RDB 快照。这使您可以在发生灾难时轻松恢复不同版本的数据集。
但是需要注意，Redis 存储 RDB 快照的时候，为了安全性的考虑，采用的是先创建一个新的 RDB 文件，然后当写入完成之后，使用新的文件去覆盖旧的 RDB 文件，因此，如果需要备份的话，可以采取这些措施：
**修改文件名**：在生成 RDB 文件时，可以通过修改 Redis 配置文件中的 dbfilename 选项为一个动态命名的文件（例如使用时间戳），这样每次快照都会生成一个新文件而不是覆盖原文件。
- 例如，可以在配置文件中设置为：`dbfilename dump-$(date +%Y%m%d%H%M%S).rdb`。不过这种方法需要在 Redis 启动时就进行设置。
**使用外部工具**：定期备份 RDB 文件到其他位置，比如使用定时任务将 dump.rdb 文件复制到其他目录，并为文件添加时间戳。
### 快速的灾难恢复和启动
RDB 的紧凑性，为它带来了这样的好处：
```
RDB is very good for disaster recovery, being a single compact file that can be transferred to far data centers, or onto Amazon S3 (possibly encrypted).
```
作为一个紧凑的文件，RDB 非常适合灾难恢复，可以传输到远数据中心或 Amazon S3（一种对象存储服务）。

```
RDB allows faster restarts with big datasets compared to AOF.
```
与 AOF 相比，RDB 允许更快地重新启动大型数据集。
## RDB 的缺点
```
RDB is NOT good if you need to minimize the chance of data loss in case Redis stops working (for example after a power outage). You can configure different _save points_ where an RDB is produced (for instance after at least five minutes and 100 writes against the data set, you can have multiple save points). However you'll usually create an RDB snapshot every five minutes or more, so in case of Redis stopping working without a correct shutdown for any reason you should be prepared to lose the latest minutes of data.
```
因为 RDB 的存储并不是实时进行的，所以，它无法保证在断电之后的完全恢复，只能恢复到已经存储快照的那个时间点上；
如果需要在 Redis 停止工作时（例如在停电后）最大限度地减少数据丢失的可能性，那 RDB 是无法做到的，因此，如果 Redis 因任何原因停止工作而没有正确关闭，最近几分钟的数据就会丢失。

```
RDB needs to fork() often in order to persist on disk using a child process. fork() can be time consuming if the dataset is big, and may result in Redis stopping serving clients for some milliseconds or even for one second if the dataset is very big and the CPU performance is not great. AOF also needs to fork() but less frequently and you can tune how often you want to rewrite your logs without any trade-off on durability.
```
虽然 RDB 使用了子进程做到了异步存储，但是每次创建 RDB 快照都需要 fork 一个子进程（fork 是同步的，可能会阻塞其他客户端），相比于 AOF，RDB 执行 fork 的频率显然是非常高的。而如果数据集非常大，或者服务器的 CPU 性能不是很强，fork 一个子进程可能会耽误几毫秒的时间。
如果确实是服务器 CPU 性能较差，或者数据集过大，可以稍微降低一下生成 RDB 的频率，使用 AOF，因为 AOF 虽然也需要 fork，但是频率相较于 RDB 是低很多的。


