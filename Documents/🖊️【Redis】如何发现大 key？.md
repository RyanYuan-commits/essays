### redis-cli --bigkeys
给出每种数据结构Top 1 bigkey。同时给出每种数据类型的键值个数＋平均大小
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
### MEMORY USAGE + SCAN
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
### redis大key检测平台
改写redis客户端，在sdk中加入埋点，实时上报数据给redis大key检测平台，监控告警。 
