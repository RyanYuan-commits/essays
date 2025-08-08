### 1 评判标准
在一般的业务场景下（并发和容量要求都不大）：单个string的 value 大于 1MB，容器元素超过 10000
在高并发且低延迟的场景：单个 string 的 value 大于 10KB，容器元素超过 5000 或整体 value 大于 10MB
阿里云 Redis：string 的 key 为 5MB，容器元素超过 10000 个，Key 中成员的数据量过大
### 2 Big Key 场景
大 key 的产生往往是业务方设计不合理，没有预见 value 的动态增长问题。  
- 一直往 value 塞数据，没有删除及过期机制
- 数据没有合理做分片，将大 key 变成以一个个小 key
接下来我们看看几类比较经典的场景：  
→ 社交类：如果某些明星或者大 v 的粉丝列表不精心设计下，必是 bigkey。  
→ 统计类：例如统计某游戏活动玩家用户的榜单列表，除非没几个人完，否则必是 bigkey。  
→ 缓存类：将数据从数据库 load 出来序列化放到 Redis 里，这个方式非常常用，但有两个地方需要注意：  
	第一，是不是有必要把所有字段都缓存  
	第二，有没有相关联的数据，关联数据分开存储
### 3 Big Key 带来的问题
- 对持久化的影响：在使用 Always 策略的时候，主线程在执行完命令后，会把数据写入到 AOF 日志文件，然后会调用 `fsync()` 函数，将内核缓冲区的数据直接写入到硬盘，等到硬盘写操作完成后，该函数才会返回；如果此时写入是一个大 Key，主线程在执行 `fsync()` 函数的时候，阻塞的时间会比较久，因为当写入的数据量很大的时候，数据同步到硬盘这个过程是很耗时的.
- 客户端超时阻塞：由于 Redis 单线程的特性，操作大 key 通常比较耗时，也就意味着阻塞 Redis 的可能性大，这样会造成客户端阻塞或者引起故障切换，会出现各种 Redis 的慢查询；
- 内存空间不均匀：集群模式在 slot 分片均匀的情况下，会出现数据和查询倾斜的情况，部分有大 key 大 Redis 节点占用内存多，QPS 高；
- 引发网络阻塞：每次获取大 key 产生的网络流量比较大，如果一个 key 大小为 1MB，每秒访问 1000 次，那每秒会产生 1000MB 的流量，这对于普通千兆网卡的服务器来说是灾难性的；
- 阻塞工作线程：执行大 key 删除的时候，在低版本 Redis 中可能会阻塞线程。
### 4 查找 Big Key
#### 4.1 Redis 自带命令
使用 `redis-cli --bigkeys` 命令，给出每种数据结构 Top 1 bigkey。同时给出每种数据类型的键值个数＋平均大小
但是无法给出查询范围，比如想查询大于10kb的所有key，--bigkeys参数就无能为力了，需要用到memory usage来计算每个键值的字节数。
```c
root@fc04fe04ddb0:/data# redis-cli --bigkeys

# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).

100.00% ||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||
Keys sampled: 1000003

-------- summary -------

Total key length in bytes is 6888916 (avg len 6.89)

Biggest string found "k1000000" has 8 bytes

0 lists with 0 items (00.00% of keys, avg size 0.00)
0 hashs with 0 fields (00.00% of keys, avg size 0.00)
0 streams with 0 entries (00.00% of keys, avg size 0.00)
1000003 strings with 6888899 bytes (100.00% of keys, avg size 6.89)
0 sets with 0 members (00.00% of keys, avg size 0.00)
0 zsets with 0 members (00.00% of keys, avg size 0.00)
```
#### 4.2 MEMORY USAGE + SCAN
文档：[https://redis.io/docs/latest/commands/memory-usage/](https://redis.io/docs/latest/commands/memory-usage/)
MEMORY USAGE：计算每个 KEY 的大小：
```c
127.0.0.1:6379> MEMORY USAGE k1000000
(integer) 72
```

SCAN：遍历 key
```c
SCAN cursor [MATCH pattern] [COUNT count] [TYPE type]
```
- cursor - 游标。
- pattern - 匹配的模式。
- count - 指定从数据集里返回多少元素，默认值为 10 。
#### 4.3 redis大key检测平台
改写redis客户端，在sdk中加入埋点，实时上报数据给redis大key检测平台，监控告警。 
### 5 Big Key 删除策略
阿里 Redis 开发规范：[https://developer.aliyun.com/article/1009125](https://developer.aliyun.com/article/1009125)
#### 5.1 string 字符串
对于 string，一般用 `DEL`，过大的话，可以使用 `UNLINK`。
#### 5.2 hash 哈希
对于 hash，每次通过 `HSCAN` 获取少量的 `FIELD`，然后通过 `HDEL` 命令来逐步删除。
- `HSCAN key cursor [MATCH pattern] [COUNT count]`
```java
public void delBigHash(String host, int port, String password, String bigHashKey) {
    Jedis jedis = new Jedis(host, port);
    if (password != null && !"".equals(password)) {
        jedis.auth(password);
    }
    ScanParams scanParams = new ScanParams().count(100);
    String cursor = "0";
    do {
        ScanResult<Entry<String, String>> scanResult = jedis.hscan(bigHashKey, cursor, scanParams);
        List<Entry<String, String>> entryList = scanResult.getResult();
        if (entryList != null && !entryList.isEmpty()) {
            for (Entry<String, String> entry : entryList) {
                jedis.hdel(bigHashKey, entry.getKey());
            }
        }
        cursor = scanResult.getStringCursor();
    } while (!"0".equals(cursor));
    //删除bigkey
    jedis.del(bigHashKey);
}
```
#### 5.3 list 列表
对于 list，使用 `LTRIM` 来逐渐删除：
- `LTRIM key start stop`，其中 `[start, stop]` 指的是要保留的范围，如果是保留到结尾的话， stop 指定为 -1。
```java
public void delBigList(String host, int port, String password, String bigListKey) {
    Jedis jedis = new Jedis(host, port);
    if (password != null && !"".equals(password)) {
        jedis.auth(password);
    }
    long llen = jedis.llen(bigListKey);
    int counter = 0;
    int left = 100;
    while (counter < llen) {
        //每次从左侧截掉100个
        jedis.ltrim(bigListKey, left, llen);
        counter += left;
    }
    //最终删除key
    jedis.del(bigListKey);
}
```
#### 5.4 set 集合
对于 set，可以使用 `SSCAN` + `SREM` 的方式删除：
```java
public void delBigSet(String host, int port, String password, String bigSetKey) {
    Jedis jedis = new Jedis(host, port);
    if (password != null && !"".equals(password)) {
        jedis.auth(password);
    }
    ScanParams scanParams = new ScanParams().count(100);
    String cursor = "0";
    do {
        ScanResult<String> scanResult = jedis.sscan(bigSetKey, cursor, scanParams);
        List<String> memberList = scanResult.getResult();
        if (memberList != null && !memberList.isEmpty()) {
            for (String member : memberList) {
                jedis.srem(bigSetKey, member);
            }
        }
        cursor = scanResult.getStringCursor();
    } while (!"0".equals(cursor));
    //删除bigkey
    jedis.del(bigSetKey);
}
```
#### 5.5 zset 有序集合
使用 `ZSCAN` + `ZREM` 的方式删除
```java
public void delBigZset(String host, int port, String password, String bigZsetKey) {
    Jedis jedis = new Jedis(host, port);
    if (password != null && !"".equals(password)) {
        jedis.auth(password);
    }
    ScanParams scanParams = new ScanParams().count(100);
    String cursor = "0";
    do {
        ScanResult<Tuple> scanResult = jedis.zscan(bigZsetKey, cursor, scanParams);
        List<Tuple> tupleList = scanResult.getResult();
        if (tupleList != null && !tupleList.isEmpty()) {
            for (Tuple tuple : tupleList) {
                jedis.zrem(bigZsetKey, tuple.getElement());
            }
        }
        cursor = scanResult.getStringCursor();
    } while (!"0".equals(cursor));
    //删除bigkey
    jedis.del(bigZsetKey);
}
```

