### 什么是深分页问题？
大家都写过分页查询，通过mysql的limit关键字，例如我要查第一页10条，那么就是limit 0,10。这看起来没啥问题。
但假如数据量很大，页数很多，我查第1000000页的10条，那就是limit 1000000,10；此时执行速度明显变慢，这就是深分页问题造成的。
同样的查询数据量，深分页可能1s左右，但是你查最初的分页的时候，可能只需要几毫秒。
### 深分页的执行流程
```sql
SELECT id, name FROM red_pakcet_detail WHERE create_time > '2022-06-13' LIMIT 100000, 10;
```
1、假设我们的 create_time 是一个二级索引，我们先要找到所有满足记录的条件，拿到了非聚集索引上面记录的主键 id。 
2、到聚集索引进行 **回表**，查询数据。 
3、扫描我们要查询的limit数据，从0开始不断扫描。最后 **抛弃** 前 100000 条。
原因：
1、limit 要扫描 10000010 条数据，并且进行丢弃。 
2、扫描更多数据也意味着回表的数据更多。
### 解决方案
#### 子查询优化
```sql
SELECT rpd.id, rpd.name
FROM red_packet_detail rpd
JOIN (
    SELECT a.id 
    FROM red_packet_detail a
    WHERE a.create_time >= '2022-06-13'
    LIMIT 100000, 10
) subquery ON rpd.id = subquery.id;
```
查询出所有需要列出的数据的 id，减少在查询过程中的 100000 次回表操作。
#### 偏移法
```sql
SELECT id, name 
FROM red_packet_detail 
WHERE id > 100000 ORDER BY id LIMIT 10;
```
从上次查询的结束位置开始查询，但是需要知道上次查询的结束位置，要求 id 自增，而且使用场景也有限制。