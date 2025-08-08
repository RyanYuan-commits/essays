### 1 案例准备
```bash
# 生成脚本
for ((i=1; i<=100*10000; i++)); do echo "set k$i v$i" >> ./redisTest.txt; done

# 执行命令
cat ./redisTest.txt | redis-cli --pipe
```
### 2 真实案例
>据云头条报道，某公司技术部发生 2 起本年度 PO 级特大事故，造成公司资金损失 400 万，原因如下：
>由于 PHP 工程师直接操作上线 redis，执行 keys wxdb（此处省略）cf8 这样的命令，导致redis锁住，导致 CPU 飙升，引起所有支付链路卡住，等十几秒结束后，所有的请求流量全部挤压到了 redis 数据库中，使数据库产生了雪崩效应，发生了数据库宕机事件。

keys * 这个指令有致命的弊端，在实际环境中最好不要使用，这个指令没有offset、limit 参数，是要一次性吐出所有满足条件的key，由于redis,是单线程的，其所有操作都是原子的，而keys算法是遍历算法，复杂度是O(n)，如果实例中有千万级以上的 key，这个指令就会导致Redis服务卡顿，所有读写Redis 的其它的指令都会被延后甚至会超时报错，可能会引起缓存雪崩甚至数据库宕机。

生产上应该限制这种命令的使用，在 `redis.conf` 的 Security 分支下，可以找到这一部分：
```c
# Command renaming (DEPRECATED).
#
# ------------------------------------------------------------------------
# WARNING: avoid using this option if possible. Instead use ACLs to remove
# commands from the default user, and put them only in some admin user you
# create for administrative purposes.
# ------------------------------------------------------------------------
#
# It is possible to change the name of dangerous commands in a shared
# environment. For instance the CONFIG command may be renamed into something
# hard to guess so that it will still be available for internal-use tools
# but not available for general clients.
#
# Example:
#
rename-command CONFIG flushdb b840fc02d524045429941cc15f59e41cb7be6c52
#
# It is also possible to completely kill a command by renaming it into
# an empty string:
# rename-command CONFIG ""
rename-command CONFIG keys ""
#
# Please note that changing the name of commands that are logged into the
# AOF file or transmitted to replicas may cause problems.
```
可以将 keys 命令配置为空字符串，此时再去客户端中调用这个命令，就会显示 ERR unknown command 'keys'。
但是在 Redis 的最新版本中，这种方式已经被废弃（不建议使用），Redis 更推荐我们使用 ZCL 的方式去管理每个用户的权限：
![[command renaming deprecated.png|600]]
比如可以通过这样的方式来创建一个只允许执行 get 和 set 方法的用户：
```
ACL SETUSER alice on >password ~* +get +set
```
这跳命令的含义是这样的：
- on：启用用户 alice。
- >password：设置 alice 的密码（这里的 password 是占位符，你可以替换为你想要的实际密码）。
- ~\*：允许用户 alice 访问所有键（~ 后面跟随键的模式）。
- +get +set：授权 alice 使用 GET 和 SET 命令。
如果不允许使用 set 命令，就可以设置为 -set。
### 3 SCAN 指令
#### 3.1 什么是 SCAN 指令？
Redis [SCAN](https://redis.com.cn/commands/scan.html) 命令及其相关命令 [SSCAN](https://redis.com.cn/commands/sscan.html)、[HSCAN](https://redis.com.cn/commands/hscan.html)、 [ZSCAN](https://redis.com.cn/commands/zscan.html) 命令都是用于增量遍历集合中的元素。
- [SCAN](https://redis.com.cn/commands/scan.html) 用于遍历当前数据库中的键。
- [SSCAN](https://redis.com.cn/commands/sscan.html) 用于遍历集合键中的元素。
- [HSCAN](https://redis.com.cn/commands/hscan.html) 用于遍历哈希键中的键值对。
- [ZSCAN](https://redis.com.cn/commands/zscan.html) 用于遍历有序集合中的元素（包括元素成员和元素分值）。
#### 3.2 SCAN 指令格式
```c
SCAN cursor [MATCH pattern] [COUNT count] [TYPE type]
```
- cursor - 游标。
- pattern - 匹配的模式。
- count - 指定从数据集里返回多少元素，默认值为 10 。
#### 3.3 基本用法
什么是 Redis 增量遍历？[SCAN](https://redis.com.cn/commands/scan.html) 命令是一个基于游标的遍历器，每次被调用之后， 都会向用户返回一个新的游标， 用户在下次遍历时需要使用这个新游标作为 [SCAN](https://redis.com.cn/commands/scan.html) 命令的游标参数， 以此来延续之前的遍历过程。
[SCAN](https://redis.com.cn/commands/scan.html) 返回一个包含两个元素的数组， 第一个元素是用于进行下一次遍历的新游标， 而第二个元素则是一个数组， 这个数组中包含了所有被遍历的元素。当 [SCAN](https://redis.com.cn/commands/scan.html) 命令的游标参数被设置为 `0` 时， 服务器将开始一次新的遍历，而当服务器向用户返回值为 `0` 的游标时， 表示遍历已结束。例如：
```
redis 127.0.0.1:6379> scan 0
1) "17"
2)  1) "key:12"
    2) "key:8"
    3) "key:4"
    4) "key:14"
    5) "key:16"
    6) "key:17"
    7) "key:15"
    8) "key:10"
    9) "key:3"
   10) "key:7"
   11) "key:1"
redis 127.0.0.1:6379> scan 17
1) "0"
2) 1) "key:5"
   2) "key:18"
   3) "key:0"
   4) "key:2"
   5) "key:19"
   6) "key:13"
   7) "key:6"
   8) "key:9"
   9) "key:11"
```